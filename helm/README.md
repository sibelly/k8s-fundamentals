### Install kind cluster

```
chmod +x kind-with-regidtry.sh
./kind-with-registry.sh
```
This script creates a kind cluster called `kind-kind-keycloak`

### Add NGINX Ingress

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/kind/deploy.yaml
```

### Install keycloak with helm

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm search repo bitnami
helm install keycloak bitnami/keycloak --version 24.4.9 -n keycloak --create-namespace -f values.yaml

```