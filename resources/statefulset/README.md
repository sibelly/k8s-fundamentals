```
kubectl apply -f nginx-statefulset.yaml
kubectl get statefulset
kubectl describe statefulset nginx
kubectl get pods

kubectl apply -f nginx-headless-service.yaml
kubectl get service
kubectl describe service nginx


kubectl run -it --rm debug --image=busybox --restart=Never -- sh
nslookup nginx-0.nginx.default.svc.cluster.local
wget -O- http://<endereço-ip-do-pod>
echo "Pod 0" > /usr/share/nginx/html/index.html
wget -O- http://<endereço-ip-do-pod>
```

A saída do comando deve ser:
```
Connecting to <endereço-ip-do-pod>:80 (<endereço-ip-do-pod>:80)
Pod 0
```

### Delete
```
kubectl delete statefulset nginx
kubectl delete -f nginx-statefulset.yaml

kubectl delete service nginx
kubectl delete -f nginx-headless-service.yaml
kubectl delete pvc www-0
```