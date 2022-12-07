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












