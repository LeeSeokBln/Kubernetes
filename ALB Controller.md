클러스터에 대한 자격 증명 공급자를 생성
```
eksctl utils associate-iam-oidc-provider \
    --region ap-northeast-2 \
    --cluster <CLUSTER NAME> \
    --approve
```
ALB Controller에 부여할 IAM 정책 생성
```
kubectl apply -k github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master
```
```
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.3.1/docs/install/iam_policy.json
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

ALB Controller에 대한 ServiceAccount 생성
```
    eksctl create iamserviceaccount \
    --cluster=<CLUSTER NAME> \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=arn:aws:iam::<ACCOUNT ID>:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --approve
```
클러스터에 ALB Controller 추가
```
wget https://github.com/jetstack/cert-manager/releases/download/v1.5.4/cert-manager.yaml
```
```
kubectl apply -f cert-manager.yaml --validate=false
```
```
curl -Lo ALB.yaml https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.4.4/v2_4_4_full.yaml
```
```
sed -i.bak -e '480,488d' ./ALB.yaml
sed -i.bak -e 's|your-cluster-name|<CLUSTER NAME>|' ./ALB.yaml
```
```
kubectl apply -f ALB.yaml
kubectl apply -f https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.4.4/v2_4_4_ingclass.yaml
```
추가된 ALB Controller 확인
```
kubectl get deployment -n kube-system aws-load-balancer-controller
```