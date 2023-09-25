Helm 설치
```
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```
```
helm repo add eks https://aws.github.io/eks-charts
```
```
helm repo update
```
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<cluster_name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller 
```

로그 확인
```
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```
