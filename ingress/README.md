### Create kind cluster with ingress

```
kind create cluster --config kind-with-ingress.yaml
```

### Installing Ingress Controller
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/kind/deploy.yaml
```

```
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

No comando acima, estamos esperando que os pods do Ingress Controller estejam prontos, com o label app.kubernetes.io/component=controller, no namespace ingress-nginx, e caso não estejam prontos em 90 segundos, o comando irá falhar.

### Create ingress resources

```
kubectl apply -f ingress-1.yaml
kubectl get ingress
kubectl describe ingress ingress-1

kubectl get ingress ingress-1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
kubectl get ingress ingress-1 -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

```

#### Testing
```
curl ADDRESS_INGRESS/giropops-senhas
```

