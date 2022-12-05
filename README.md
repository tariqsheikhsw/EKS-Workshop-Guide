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



