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


Reference:

https://aws.amazon.com/blogs/storage/running-wordpress-on-amazon-eks-with-amazon-efs-intelligent-tiering/

https://www.eksworkshop.com/intermediate/250_cloudwatch_container_insights/installwordpress/

https://www.stacksimplify.com/aws-eks/kubernetes-storage/aws-eks-storage-with-aws-rds-database/


Create RDS DB 
```

kubectl apply -f 01-MySQL-externalName-Service.yml

kubectl run -it --rm --image=mysql:5.7.22 --restart=Never mysql-client -- mysql -h usermgmtdb.cbzqfdsbhxpu.us-west-2.rds.amazonaws.com -u dbadmin -pdbpassword11

show schemas;
create database usermgmt;
show schemas;
exit

kubectl apply -f .


```
https://www.stacksimplify.com/aws-eks/kubernetes-storage/aws-eks-storage-with-aws-rds-database/



WAF
```
WAF_AWS_REGION=us-west-2
WAF_ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
WAF_EKS_CLUSTER_NAME=eksworkshop-eksctl
```

```
WAF_VPC_ID=$(aws eks describe-cluster \
  --name $WAF_EKS_CLUSTER_NAME \
  --region $WAF_AWS_REGION \
  --query 'cluster.resourcesVpcConfig.vpcId' \
  --output text)
```

```
eksctl utils associate-iam-oidc-provider \
  --cluster $WAF_EKS_CLUSTER_NAME \
  --region $WAF_AWS_REGION \
  --approve.
```

```
curl -S https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/install/iam_policy.json -o iam-policy.json 

WAF_LBC_IAM_POLICY_ARN=$(aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy-WAFDEMO \
  --policy-document file://iam-policy.json \
  --query 'Policy.Arn' \
  --output text)
  
eksctl create iamserviceaccount \
  --cluster=$WAF_EKS_CLUSTER_NAME \
  --region $WAF_AWS_REGION \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --override-existing-serviceaccounts \
  --attach-policy-arn=arn:aws:iam::${WAF_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy-WAFDEMO \
  --approve
  
helm repo add eks https://aws.github.io/eks-charts && helm repo update
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
helm install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=$WAF_EKS_CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set vpcId=$WAF_VPC_ID \
  --set region=$WAF_AWS_REGION
```
```
git clone https://github.com/aws/aws-app-mesh-examples.git
cd aws-app-mesh-examples/walkthroughs/eks-getting-started/
kubectl apply -f infrastructure/yelb_initial_deployment.yaml
```
```
cat << EOF > yelb-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: yelb.app
  namespace: yelb
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: yelb-ui
                port:
                  number: 80
EOF
kubectl apply -f yelb-ingress.yaml 
```

WireGuard / Cilium CNI (POD to POD encryption)

https://aws.amazon.com/blogs/containers/transparent-encryption-of-node-to-node-traffic-on-amazon-eks-using-wireguard-and-cilium/

```
export AWS_REGION=us-west-2


cat << EOF > clusterconfig.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: wireguard-blog
  region: $AWS_REGION

iam:
  withOIDC: true
  
addons:
- name: vpc-cni

nodeGroups:
- name: bottlerocket
  instanceType: t3.medium
  desiredCapacity: 2
  amiFamily: Bottlerocket
  iam:
    attachPolicyARNs:
    - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
    - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
    - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
EOF
 
eksctl create cluster -f clusterconfig.yaml
```

```
helm repo add cilium https://helm.cilium.io/
helm repo update
```
```
helm install cilium cilium/cilium --version 1.12.2 \
  --namespace kube-system \
  --set cni.chainingMode=aws-cni \
  --set enableIPv4Masquerade=false \
  --set tunnel=disabled \
  --set endpointRoutes.enabled=true \
  --set encryption.enabled=true \
  --set encryption.type=wireguard \
  --set l7Proxy=false 
```

```
kubectl -n kube-system exec -it ds/cilium -- cilium status | grep Encryption 
```

```
cat << EOF > server-pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: server
  labels:
    blog: wireguard
    name: server
spec:
  containers:
    - name: server
      image: nginx
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: "kubernetes.io/hostname"
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        blog: wireguard
---
apiVersion: v1
kind: Service
metadata:
  name: server
spec:
  selector:
    name: server
  ports:
  - port: 80
EOF

kubectl apply -f server-pod.yaml
```

```
cat << EOF > client-pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: client
  labels:
    blog: wireguard
    name: client
spec:
  containers:
    - name: client
      image: busybox
      command: ["watch", "wget", "server"]
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: "kubernetes.io/hostname"
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        blog: wireguard
EOF

kubectl apply -f client-pod.yaml
```
```
kubectl get pod -o wide
```

Primary Region/Cloud9 : us-west-2
Two EKS clusters: us-west-2 , us-east-2
ECR repository (votingapp) : us-west-2 , us-east-2
DynamoDB global table (votingapp)

EKS Cluster Names : primary, secondary
```
export AWS_REGION_1=us-west-2
export AWS_REGION_2=us-east-2
export EKS_CLUSTER_1=primary
export EKS_CLUSTER_2=secondary
export my_domain=cloud4life.io
export ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
```

```
aws eks update-kubeconfig --name primary --region ${AWS_REGION_1}
aws eks update-kubeconfig --name secondary --region ${AWS_REGION_2}
```
```
kubectl get nodes 
```

```
eksctl utils associate-iam-oidc-provider \
  --region $AWS_REGION_1 \
  --cluster $EKS_CLUSTER_1 \
  --approve
```
```
eksctl utils associate-iam-oidc-provider \
  --region $AWS_REGION_2 \
  --cluster $EKS_CLUSTER_2 \
  --approve
```



Karpenter
```
https://karpenter.sh/preview/getting-started/getting-started-with-eksctl/

export KARPENTER_VERSION=v0.20.0

export CLUSTER_NAME="eks-karpenter-demo"
export AWS_DEFAULT_REGION="us-west-2"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"

echo $KARPENTER_VERSION $CLUSTER_NAME $AWS_DEFAULT_REGION $AWS_ACCOUNT_ID

eksctl create cluster -f - << EOF
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_DEFAULT_REGION}
  version: "1.23"
  tags:
    karpenter.sh/discovery: ${CLUSTER_NAME}
managedNodeGroups:
  - instanceType: m5.large
    amiFamily: AmazonLinux2
    name: ${CLUSTER_NAME}-ng
    desiredCapacity: 2
    minSize: 1
    maxSize: 10
iam:
  withOIDC: true
EOF

export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.endpoint" --output text)"




```


Demonstrate Consolidation within EKS (Karpenter)
https://www.youtube.com/watch?v=OB7IZolZk78
https://ec2spotworkshops.com/karpenter/050_karpenter/consolidation.html
https://karpenter.sh/preview/getting-started/getting-started-with-eksctl/

```
kubectl apply -f inflate.yaml
!
kubectl scale deployment inflate --replicas=60
!
kubectl scale deployment inflate --replicas=10
!

```

```
https://github.com/awslabs/eks-node-viewer
```
```
Path: /home/ec2-user/go/bin/

eks-node-viewer

eks-node-viewer --nodeSelector "karpenter.sh/provisioner-name"

eks-node-viewer --resources cpu,memory
```

### Deploying Applications in Kubernetres ###

### Step by Step Guide

### Deploy a Stateless Application in a Kubernetes Cluster
The application runs in a client web browser and doesn't store any state across sessions

1. Create a Deployment File
2. Create a Service Resource File 
3. [Optional] Scale Deployment (replicas) 


kubectl create -f deployment.yml
kubectl get deployment game-deployment
kubectl describe deployment game-deployment
kubectl get events | more
kubectl get pods
kubectl describe pods
kubectl get pods -l app=game
!

```
cat <<EOF > deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: game-deployment
spec:
  # tells deployment to run 1 pods matching the template
  replicas: 1
  # create pods using the pod definition in this template
  selector:
    matchLabels:
      app: game
  template:
    metadata:
      # name is automatically generated based on the deployment.name
      labels:
        app: game
    spec:
      containers:
      - name: tetris
        image: lrakai/tetris:latest
        ports:
        - containerPort: 80
EOF
```

kubectl create -f service.yml
kubectl describe services game
ssh worker1 -o StrictHostKeyChecking=no "curl -s ifconfig.me; echo"

```
cat <<EOF > service.yml
apiVersion: v1
kind: Service
metadata:
  name: game
  labels:
    app: game
spec:
  selector:
    # Use labels to select the pods to route traffic to
    app: game
  ports:
  - protocol: TCP
    port: 80
  # Allocate a port on each node in the cluster
  type: NodePort
EOF
```

Scaling 


```
sed -i 's/\(replicas: \).*/\12/' deployment.yml
kubectl apply -f deployment.yml
kubectl get pods -l app=game
```

### Deploy a Stateful Application in a Kubernetes Cluster
![image](https://user-images.githubusercontent.com/54164634/208092618-df5cda62-32f4-40d8-9f3e-75495330eff2.png)

Stateful applications are applications that have a memory of what happened in the past. Databases are an example of stateful applications. Kubernetes provides support for stateful applications through StatefulSets and related primitives.

1. Create a Config Map 
2. Create Services for MySQL application (Headless Service for mysql and ClusterIP type service for mysql-reads)
3. Declare a default Storage Class that will be used to dynamically provision general-purpose (gp2) EBS volumes for the Kubernetes PVs
4. Declare the StatefulSet for MySQL 
5. Run temporary container to test services and Service Exposing StatefulSet Via Load Balancer

```
ssh ubuntu@X.X.X.X -oStrictHostKeyChecking=no
watch kubectl get nodes
source <(kubectl completion bash)
kubectl describe nodes -l node-role.kubernetes.io/control-plane | more
kubectl get namespace
kubectl get pods --namespace=kube-system

```

All of the components of the cluster are running in pods in the kube-system namespace:

    calico: The container network used to connect each node to every other node in the cluster. Calico also supports network policy. Calico is one of many possible container networks that can be used by Kubernetes.
    coredns: Provides DNS services to nodes in the cluster
    etcd: The primary data store of all cluster state
    kube-apiserver: The REST API server for managing the Kubernetes cluster
    kube-controller-manager: Manager of all of the controllers in the cluster that monitor and change the cluster state when necessary
    kube-proxy: Network proxy that runs on each node
    kube-scheduler: Control plane process which assigns Pods to Nodes
    metrics-server: Not an essential component of a Kubernetes cluster but it is used in this lab to provide metrics for viewing in the Kubernetes dashboard.
    ebs-csi: Not an essential component of a Kubernetes cluster but is used to manage the lifecycle of Amazon EBS volumes for persistent volumes.
    
 
ConfigMaps: A type of Kubernetes resource that is used to decouple configuration artifacts from image content to keep containerized applications portable. The configuration data is stored as key-value pairs.

 

Headless Service: A headless service is a Kubernetes service resource that won't load balance behind a single service IP. Instead, a headless service returns a list of DNS records that point directly to the pods that back the service. A headless service is defined by declaring the clusterIP property in a service spec and setting the value to None. StatefulSets currently require a headless service to identify pods in the cluster network.

 

Stateful Sets: Similar to Deployments in Kubernetes, StatefulSets manage the deployment and scaling of pods given a container spec.StatefulSets differ from Deployments in that the Pods in a stateful set are not interchangeable. Each pod in a StatefulSet has a persistent identifier that it maintains across any rescheduling. The pods in a StatefulSet are also ordered. This provides a guarantee that one pod can be created before following pods. In this Lab, this is useful for ensuring the MySQL primary is provisioned first.

 

PersistentVolumes (PVs) and PersistentVolumeClaims (PVCs): PVs are Kubernetes resources that represent storage in the cluster. Unlike regular Volumes which exist only until while containing pod exists, PVs do not have a lifetime connected to a pod. Thus, they can be used by multiple pods over time, or even at the same time. Different types of storage can be used by PVs including NFS, iSCSI, and cloud-provided storage volumes, such as AWS EBS volumes. Pods claim PV resources through PVCs.

 

MySQL replication: This Lab uses a single primary, asynchronous replication scheme for MySQL. All database writes are handled by a single primary. The database replicas asynchronously synchronize with the primary. This means the primary will not wait for the data to be copied onto the replicas. This can improve the performance of the primary at the expense of having replicas that are not always exact copies of the primary. Many applications can tolerate slight differences in the data and are able to improve the performance of database read workloads by allowing clients to read from the replicas.


Config Map
```
cat <<EOF > mysql-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  labels:
    app: mysql
data:
  master.cnf: |
   # Apply this config only on the primary.
   [mysqld]
   log-bin
  slave.cnf: |
    # Apply this config only on replicas.
    [mysqld]
    super-read-only
EOF
```

kubectl create -f mysql-configmap.yaml

MySQL Services (Headless and ClusterIP Type)
```
cat <<EOF > mysql-services.yaml
# Headless service for stable DNS entries of StatefulSet members.
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
---
# Client service for connecting to any MySQL instance for reads.
# For writes, you must instead connect to the primary: mysql-0.mysql.
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
EOF
```
kubectl create -f mysql-services.yaml


Storage Class
```
cat <<EOF > mysql-storageclass.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: general
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
EOF
```

kubectl create -f mysql-storageclass.yaml


StatefulSet / MySQL
```
cat <<'EOF' > mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:5.7.35
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Generate mysql server-id from pod ordinal index.
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # Add an offset to avoid reserved server-id=0 value.
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # Copy appropriate conf.d files from config-map to emptyDir.
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/slave.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      - name: clone-mysql
        image: gcr.io/google-samples/xtrabackup:1.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Skip the clone if data already exists.
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          # Skip the clone on primary (ordinal index 0).
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] && exit 0
          # Clone data from previous peer.
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # Prepare the backup.
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: "1"
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          timeoutSeconds: 5
        readinessProbe:
          exec:
            # Check we can execute queries over TCP (skip-networking is off).
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          timeoutSeconds: 1
      - name: xtrabackup
        image: gcr.io/google-samples/xtrabackup:1.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql
          # Determine binlog position of cloned data, if any.
          if [[ -f xtrabackup_slave_info ]]; then
            # XtraBackup already generated a partial "CHANGE MASTER TO" query
            # because we're cloning from an existing replica.
            mv xtrabackup_slave_info change_master_to.sql.in
            # Ignore xtrabackup_binlog_info in this case (it's useless).
            rm -f xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # We're cloning directly from primary. Parse binlog position.
            [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm xtrabackup_binlog_info
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi
          # Check if we need to complete a clone by starting replication.
          if [[ -f change_master_to.sql.in ]]; then
            echo "Waiting for mysqld to be ready (accepting connections)"
            until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done
            echo "Initializing replication from clone position"
            # In case of container restart, attempt this at-most-once.
            mv change_master_to.sql.in change_master_to.sql.orig
            mysql -h 127.0.0.1 <<EOF
          $(<change_master_to.sql.orig),
            MASTER_HOST='mysql-0.mysql',
            MASTER_USER='root',
            MASTER_PASSWORD='',
            MASTER_CONNECT_RETRY=10;
          START SLAVE;
          EOF
          fi
          # Start a server to send backups when requested by peers.
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 100m
            memory: 50Mi
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 2Gi
      storageClassName: general
EOF
```

There is a lot going on in the StatefulSet. Don't focus too much on the bash scripts that are performing MySQL-specific tasks. Some highlights to focus on, following the order they appear in the file are:

    init-containers: Run to completion before any containers in the Pod spec
        init-mysql: Assigns a unique MySQL server ID starting from 100 for the first pod and incrementing by one, as well as copying the appropriate configuration file from the config-map. Note the config-map is mounted via the VolumeMounts section. The ID and appropriate configuration file are persisted on the conf volume.
        clone-mysql: For pods after the primary, clone the database files from the preceding pod. The xtrabackup tool performs the file cloning and persists the data on the data volume.
    spec.containers: Two containers in the pod
        mysql: Runs the MySQL daemon and mounts the configuration in the conf volume and the data in the data volume
        xtrabackup: A sidecar container that provides additional functionality to the mysql container. It starts a server to allow data cloning and begins replication on replicas using the cloned data files.
    spec.volumes: conf and config-map volumes are stored on the node's local disk. They are easily re-generated if a failure occurs and don't require PVs.
    volumeClaimTemplates: A template for each pod to create a PVC with. ReadWriteOnce accessMode allows the PV to be mounted by only one node at a time in read/write mode. The storageClassName references the AWS EBS gp2 storage class named general that you created earlier. 
    
```
kubectl create -f mysql-statefulset.yaml
kubectl get pods -l app=mysql --watch
kubectl describe pv
kubectl describe pvc
kubectl get statefulset
```

In this lab step, you created several Kubernetes cluster resources to deploy the MySQL database as an example stateful application:

    A ConfigMap for decoupling primary and replica configuration from the containers
    Two Services: one headless service to manage network identity of pods in the StatefulSet, and one to load balance read access to the MySQL replicas
    A StorageClass to provision EBS PVs dynamically
    A StatefulSet that declared two init-containers, two containers, and one PVC template

You observed the ordered sequence of pods being initialized and the PVs created in AWS to facilitate the StatefulSet.


Temporary Container : 
```
kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  /usr/bin/mysql -h mysql-0.mysql -e "CREATE DATABASE mydb; CREATE TABLE mydb.notes (note VARCHAR(250)); INSERT INTO mydb.notes VALUES ('k8s Cloud Academy Lab');"
  
kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  /usr/bin/mysql -h mysql-read -e "SELECT * FROM mydb.notes"
  
kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
  bash -ic "while sleep 1; do /usr/bin/mysql -h mysql-read -e 'SELECT @@server_id'; done"
  
kubectl get pod -o wide

node=$(kubectl get pods --field-selector metadata.name=mysql-2 -o=jsonpath='{.items[0].spec.nodeName}')
kubectl drain $node --force --delete-local-data --ignore-daemonsets

kubectl get pod -o wide --watch

kubectl uncordon $node

kubectl delete pod mysql-2
kubectl get pod mysql-2 -o wide --watch

kubectl scale --replicas=5 statefulset mysql

kubectl get pods -l app=mysql --watch

kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
  bash -ic "while sleep 1; do /usr/bin/mysql -h mysql-read -e 'SELECT @@server_id'; done"
  
kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  /usr/bin/mysql -h mysql-4.mysql -e "SELECT * FROM mydb.notes"
  
kubectl get services mysql-read

echo "  type: LoadBalancer" >> mysql-services.yaml

kubectl apply -f mysql-services.yaml

kubectl get services mysql-read

kubectl describe services mysql-read | grep "LoadBalancer Ingress"

load_balancer=$(kubectl get services mysql-read -o=jsonpath='{.status.loadBalancer.ingress[0].hostname}')
kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
  bash -ic "while sleep 1; do /usr/bin/mysql -h $load_balancer -e 'SELECT @@server_id'; done"


```

