### Create aws eks cluster

```eksctl create cluster --name=eks-cluster --version=1.32 --region=us-east-1 --nodegroup-name=eks-cluster-nodegroup --node-type=t3.medium --nodes=2 --nodes-min=1 --nodes-max=3 --managed

```
