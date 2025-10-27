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

#### Keycloak can be accessed through the following DNS name from within your cluster:
```yaml
    keycloak.keycloak.svc.cluster.local (port 80)
```
To access Keycloak from outside the cluster execute the following commands:

1. Get the Keycloak URL by running these commands:
```
    export HTTP_NODE_PORT=$(kubectl get --namespace keycloak -o jsonpath="{.spec.ports[?(@.name=='http')].nodePort}" services keycloak)
    export NODE_IP=$(kubectl get nodes --namespace keycloak -o jsonpath="{.items[0].status.addresses[0].address}")

    echo "http://${NODE_IP}:${HTTP_NODE_PORT}/"
```
2. Access Keycloak using the obtained URL.
3. Access the Administration Console using the following credentials:
```
  echo Username: admin
  echo Password: $(kubectl get secret --namespace keycloak keycloak -o jsonpath="{.data.admin-password}" | base64 -d)
```

### OLM

- Remove
```
k delete -f https://raw.githubusercontent.com/operator-framework/operator-lifecycle-manager/master/deploy/upstream/quickstart/olm.yaml
```

### Things operator keycloak create
```
 kubectl apply -f https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/26.2.5/kubernetes/kubernetes.yml

serviceaccount/keycloak-operator created
clusterrole.rbac.authorization.k8s.io/keycloak-operator-clusterrole unchanged
clusterrole.rbac.authorization.k8s.io/keycloakrealmimportcontroller-cluster-role unchanged
clusterrole.rbac.authorization.k8s.io/keycloakcontroller-cluster-role unchanged
clusterrolebinding.rbac.authorization.k8s.io/keycloak-operator-clusterrole-binding unchanged
role.rbac.authorization.k8s.io/keycloak-operator-role created
rolebinding.rbac.authorization.k8s.io/keycloak-operator-role-binding created
rolebinding.rbac.authorization.k8s.io/keycloakrealmimportcontroller-role-binding created
rolebinding.rbac.authorization.k8s.io/keycloakcontroller-role-binding created
rolebinding.rbac.authorization.k8s.io/keycloak-operator-view created
service/keycloak-operator created
deployment.apps/keycloak-operator created
```

### Debug
```
 kubectl explain keycloak.spec
```