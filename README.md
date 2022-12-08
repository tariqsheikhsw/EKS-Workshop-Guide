# EKS-Workshop-Guide

Quick Spin Up in Cloud9

IAM Role
```
aws sts get-caller-identity
```

Create Cluster
```
eksctl create cluster -f eksworkshop.yaml
```

Check Nodes

```
kubectl get nodes
```

Update KubeConfig
```
aws eks update-kubeconfig --name eksworkshop-eksctl --region ${AWS_REGION}
```

Export names
```
STACK_NAME=$(eksctl get nodegroup --cluster eksworkshop-eksctl -o json | jq -r '.[].StackName')
ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name $STACK_NAME | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')
echo "export ROLE_NAME=${ROLE_NAME}" | tee -a ~/.bash_profile
```

Console Credentials
```
c9builder=$(aws cloud9 describe-environment-memberships --environment-id=$C9_PID | jq -r '.memberships[].userArn')
if echo ${c9builder} | grep -q user; then
	rolearn=${c9builder}
        echo Role ARN: ${rolearn}
elif echo ${c9builder} | grep -q assumed-role; then
        assumedrolename=$(echo ${c9builder} | awk -F/ '{print $(NF-1)}')
        rolearn=$(aws iam get-role --role-name ${assumedrolename} --query Role.Arn --output text) 
        echo Role ARN: ${rolearn}
fi
```
Create the identity mapping within the cluster
```
eksctl create iamidentitymapping --cluster eksworkshop-eksctl --arn ${rolearn} --group system:masters --username admin
```

Verify entry in the AWS auth map
```
kubectl describe configmap -n kube-system aws-auth
```

Cloning Source Repo
```
cd ~/environment
git clone https://github.com/aws-containers/eks-app-mesh-polyglot-demo.git
cd eks-app-mesh-polyglot-demo
```

Helm
```
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```
```
helm version --short
```
```
helm list
```

Deploy Helm Chart
```
cd ~/environment/eks-app-mesh-polyglot-demo
helm install --debug --dry-run workshop ~/environment/eks-app-mesh-polyglot-demo/workshop/helm-chart/
```

```
cd ~/environment/eks-app-mesh-polyglot-demo
helm install workshop ~/environment/eks-app-mesh-polyglot-demo/workshop/helm-chart/
```

```
kubectl get pod,svc -n workshop -o wide
```

Application 
```
export LB_NAME=$(kubectl get svc frontend -n workshop -o jsonpath="{.status.loadBalancer.ingress[*].hostname}") 
echo $LB_NAME
```



Persistent Storage with AWS EFS

```
CLUSTER_NAME=eksworkshop-eksctl
VPC_ID=$(aws eks describe-cluster --name $CLUSTER_NAME --query "cluster.resourcesVpcConfig.vpcId" --output text)
CIDR_BLOCK=$(aws ec2 describe-vpcs --vpc-ids $VPC_ID --query "Vpcs[].CidrBlock" --output text)
```

```
MOUNT_TARGET_GROUP_NAME="eks-efs-group"
MOUNT_TARGET_GROUP_DESC="NFS access to EFS from EKS worker nodes"
MOUNT_TARGET_GROUP_ID=$(aws ec2 create-security-group --group-name $MOUNT_TARGET_GROUP_NAME --description "$MOUNT_TARGET_GROUP_DESC" --vpc-id $VPC_ID | jq --raw-output '.GroupId')
aws ec2 authorize-security-group-ingress --group-id $MOUNT_TARGET_GROUP_ID --protocol tcp --port 2049 --cidr $CIDR_BLOCK
```

EFS File System

```
FILE_SYSTEM_ID=$(aws efs create-file-system | jq --raw-output '.FileSystemId')
```
```
aws efs describe-file-systems --file-system-id $FILE_SYSTEM_ID
```

Create Mount Target
```
TAG1=tag\:alpha.eksctl.io/cluster-name
TAG2=tag\:kubernetes.io/role/elb
subnets=($(aws ec2 describe-subnets --filters "Name=$TAG1,Values=$CLUSTER_NAME" "Name=$TAG2,Values=1" | jq --raw-output '.Subnets[].SubnetId'))
for subnet in ${subnets[@]}
do
    echo "creating mount target in " $subnet
    aws efs create-mount-target --file-system-id $FILE_SYSTEM_ID --subnet-id $subnet --security-groups $MOUNT_TARGET_GROUP_ID
done
```
```
aws efs describe-mount-targets --file-system-id $FILE_SYSTEM_ID | jq --raw-output '.MountTargets[].LifeCycleState'
```

EFS CSI Driver
```
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm repo update
helm upgrade --install aws-efs-csi-driver --namespace kube-system aws-efs-csi-driver/aws-efs-csi-driver
```
```
kubectl get pods -n kube-system
```

Creating PV 
```
sed -i "s/EFS_VOLUME_ID/$FILE_SYSTEM_ID/g" ~/environment/eks-app-mesh-polyglot-demo/workshop/efs-pvc.yaml
```

```
kubectl apply -f ~/environment/eks-app-mesh-polyglot-demo/workshop/efs-pvc.yaml
```

Verify
```
kubectl get pvc -n workshop
kubectl get pv
```

Stateful Services
```
cd ~/environment/eks-app-mesh-polyglot-demo
helm upgrade --reuse-values -f ~/environment/eks-app-mesh-polyglot-demo/workshop/helm-chart/values-efs.yaml workshop workshop/helm-chart/
```

Revert back 
```
helm upgrade --reuse-values -f ~/environment/eks-app-mesh-polyglot-demo/workshop/helm-chart/values.yaml workshop workshop/helm-chart/
```

```
export PROD_CATALOG=$(kubectl get pods -n workshop -l app=prodcatalog -o jsonpath='{.items[].metadata.name}') 
kubectl -n workshop describe pod  ${PROD_CATALOG}
```


```
export PROD_CATALOG=$(kubectl get pods -n workshop -l app=prodcatalog -o jsonpath='{.items[].metadata.name}') 
kubectl -n workshop exec -it ${PROD_CATALOG} -c prodcatalog -- /bin/bash
```

```
cat /products/products.txt 
```

```
kubectl get pod -n workshop
```
```
kubectl scale --replicas=2 deployment/prodcatalog -n workshop

kubectl get pod -n workshop
```

```
kubectl -n workshop exec -it prodcatalog-xxx-nmzmd  -c prodcatalog -- /bin/bash
```


Load Balancer Controller
```
eksctl utils associate-iam-oidc-provider \
      --region us-west-2 \
      --cluster eksworkshop-eksctl \
      --approve
```
IAM Policy
```
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.0/docs/install/iam_policy.json
```

```
aws iam create-policy \
      --policy-name AWSLoadBalancerControllerIAMPolicy \
      --policy-document file://iam-policy.json
```

Service Account

```
eksctl create iamserviceaccount \
--cluster=eksworkshop-eksctl \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::583497507745:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--region us-west-2 \
--approve
```

Deploy LB using Helm
```
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash

helm repo add eks https://aws.github.io/eks-charts


helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=eksworkshop-eksctl --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller

kubectl get pods -n kube-system

```

Ingress

IngressClass

```
cat <<EOF | kubectl create -f -
apiVersion: networking.k8s.io/v1
kind: IngressClass 
metadata:
  name: aws-alb
spec:
  controller: ingress.k8s.aws/alb #Defines the controller name which implements this IngressClass. By default AWS Load Balancer Controller uses and checks this controller name. 
EOF
```

```
kubectl get ingressclass
```

Ingress
```
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: workshop
  name: workshopingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing #Places the load balancer on public subnets
    alb.ingress.kubernetes.io/target-type: ip #The Pod IPs should be used as the target IPs (rather than the node IPs as was the case with NLB in hte previous section)
    alb.ingress.kubernetes.io/group.name: product-catalog # Groups multiple Ingress resources
spec:
  ingressClassName: aws-alb #Defines which IngessClass that this Ingress is associated with. This specific Ingress Class is owned by  AWS Load Balancer Controller. Hence it will fulfill this Ingress.
  rules:
  - http:
      paths:
      - path: /new
        pathType: Prefix
        backend:
          service:
            name: frontendnew
            port:
               number: 80
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
EOF
```
```
kubectl get ingress -n workshop
```
```
kubectl describe ingress workshopingress -n workshop
```

Deployment/Service

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontendnew
  namespace: workshop
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontendnew
  template:
    metadata:
      labels:
        app: frontendnew
    spec:
      containers:
      - image: public.ecr.aws/u2g6w7p2/eks-workshop-demo/simplewebserver:1.0
        name: simplewebserver
---
apiVersion: v1
kind: Service
metadata:
  name: frontendnew
  namespace: workshop
spec:
  type: ClusterIP
  ports:
    - port: 80 
      name: http 
  selector:
    app: frontendnew
EOF
```
```
kubectl get svc,deployment -n workshop
```
```
kubectl get pods -n workshop --selector app=frontendnew -o wide
```
```
kubectl describe service frontendnew -n workshop
```
```
kubectl describe ingress workshopingress -n workshop
```

EFS
```
CLUSTER_NAME=eksworkshop-eksctl
VPC_ID=$(aws eks describe-cluster --name $CLUSTER_NAME --query "cluster.resourcesVpcConfig.vpcId" --output text)
CIDR_BLOCK=$(aws ec2 describe-vpcs --vpc-ids $VPC_ID --query "Vpcs[].CidrBlock" --output text)
```

```
MOUNT_TARGET_GROUP_NAME="efs-mount-sg"
MOUNT_TARGET_GROUP_DESC="NFS access to EFS from EKS worker nodes"
MOUNT_TARGET_GROUP_ID=$(aws ec2 create-security-group --group-name $MOUNT_TARGET_GROUP_NAME --description "$MOUNT_TARGET_GROUP_DESC" --vpc-id $VPC_ID | jq --raw-output '.GroupId')
aws ec2 authorize-security-group-ingress --group-id $MOUNT_TARGET_GROUP_ID --protocol tcp --port 2049 --cidr $CIDR_BLOCK
```

```
FILE_SYSTEM_ID=$(aws efs create-file-system | jq --raw-output '.FileSystemId')

aws efs describe-file-systems --file-system-id $FILE_SYSTEM_ID

TAG1=tag\:alpha.eksctl.io/cluster-name
TAG2=tag\:kubernetes.io/role/elb
subnets=($(aws ec2 describe-subnets --filters "Name=$TAG1,Values=$CLUSTER_NAME" "Name=$TAG2,Values=1" | jq --raw-output '.Subnets[].SubnetId'))
for subnet in ${subnets[@]}
do
    echo "creating mount target in " $subnet
    aws efs create-mount-target --file-system-id $FILE_SYSTEM_ID --subnet-id $subnet --security-groups $MOUNT_TARGET_GROUP_ID
done


aws efs describe-mount-targets --file-system-id $FILE_SYSTEM_ID | jq --raw-output '.MountTargets[].LifeCycleState'
```


Access Point for /wordpress
```
GUI
```

EFS CSI Driver
```
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm repo update
helm upgrade --install aws-efs-csi-driver --namespace kube-system aws-efs-csi-driver/aws-efs-csi-driver
```


```
kubectl apply -k .
```








