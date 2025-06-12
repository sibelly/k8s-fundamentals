# k8s-fundamentals

- Full cycle live -> https://www.youtube.com/watch?v=oxWEVQP5_Rg

![k8s-scheme](https://github.com/sibelly/k8s-fundamentals/blob/main/resources/k8s-scheme.png?raw=true)


### Pod
Menor unidade, agrupamento de containers que compartilham do mesmo namespace (do kernel), por exemplo, endereçamento de rede.

### Tools
- [kind](https://kind.sigs.k8s.io/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)


## Create cluster with kind

```kind create cluster --name=esquenta```

```kubectl cluster-info --context kind-esquenta```

```kubectl get nodes```

## Create Pod

```kubectl apply -f pod.yaml```

```kubectl get pods```

```kubectl port-forward pod/nginx-demo 9090:80```

## Create ReplicaSet
```kubectl apply -f replicaset.yaml```

## Create Deployment
```kubectl apply -f deployment.yaml ```

### Docker basics

- Stop all processes from running containers with

```
- docker stop $(docker ps -a -q)
```

- Remove stopped containers.
```
docker rm $(docker ps -a -q)

docker rm -f some-control-plane
```
- Remove images
```
docker rmi $(docker images -q)
```

### k8s
```
kubectl config delete-cluster kind-sibops
kubectl config delete-context kind-sibops

kind create cluster --config kind-cluster.yaml --name sibops
```