# k8s-fundamentals

- Full cycle live -> https://www.youtube.com/watch?v=oxWEVQP5_Rg

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