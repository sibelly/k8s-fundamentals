### Create aws eks cluster
https://github.com/eksctl-io/eksctl

```
eksctl create cluster --name=eks-cluster --version=1.32 --region=us-east-1 --nodegroup-name=eks-cluster-nodegroup --node-type=t3.medium --nodes=2 --nodes-min=1 --nodes-max=3 --managed

eksctl delete cluster --region us-east-1 --name cluster-name

eksctl get cluster -A --region us-east-1

```
