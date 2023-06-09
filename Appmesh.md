# AWS App Mesh는 서비스가 여러 유형의 컴퓨팅 인프라에서 서로 쉽게 통신할 수 있도록 애플리케이션 수준 네트워킹을 제공하는 서비스 메시입니다*
Service mesh : service mesh 는 그 안에 상주하는 서비스 간의 네트워크 트래픽에 대한 논리적 경계입니다.
Virtual services : 가상 서비스는 가상 노드가 직접 또는 가상 라우터를 통해 간접적으로 제공하는 실제 서비스의 추상화입니다.
Virtual nodes : 가상 노드는 ECS 서비스 또는 Kubernetes 배포와 같은 특정 작업 그룹에 대한 논리적 포인터 역할을 합니다.  가상 노드를 생성할 때 작업 그룹의 서비스 검색 이름을 지정해야 합니다.
Envoy proxy : Envoy 프록시는 가상 라우터 및 가상 노드에 대해 설정한 App Mesh 서비스 메시 트래픽 규칙을 사용하도록 마이크로 서비스 작업 그룹을 구성합니다.  뿐만 아니라 가상 노드, 가상 라우터, 경로 및 가상 서비스를 생성한 후 작업 그룹에 Envoy 컨테이너를 추가해야 합니다.
Virtual routers : 가상 라우터는 메시 내에서 하나 이상의 가상 서비스에 대한 트래픽을 처리합니다.
Routes : 경로는 서비스 이름 접두사와 일치하는 트래픽을 하나 이상의 가상 노드로 보내기 위해 가상 라우터와 연결됩니다.

# 사전 준비

## EKS 클러스터에 연결하려면 Kubeconfig를 업데이트합니다.
```
aws eks --region ap-northeast-2 update-kubeconfig --name 클러스터 이름
```

## Helm에 eks-charts 리포지토리를 추가합니다
```
helm repo add eks https://aws.github.io/eks-charts
```

## App Mesh Kubernetes 사용자 지정 리소스 정의(CRD)를 설치합니다.
```
kubectl apply -k "https://github.com/aws/eks-charts/stable/appmesh-controller/crds?ref=master"
```

## appmesh-system 네임스페이스를 만듭니다.
```
kubectl create ns appmesh-system
```

## 클러스터에 대한 OIDC(OpenID Connect) 자격 증명 공급자를 생성합니다.
```
eksctl utils associate-iam-oidc-provider --region=ap-northeast-2 --cluster 클러스터 이름 --approve
```

## IAM 역할을 생성하고 AWSAppMeshFullAccess 및 AWSCloudMapFullAccess 정책을 연결하고 appmesh-controller Kubernetes 서비스 계정에 바인딩합니다.
```
eksctl create iamserviceaccount \
    --cluster 클러스터 이름 \
    --namespace appmesh-system \
    --name appmesh-controller \
    --attach-policy-arn  arn:aws:iam::aws:policy/AWSCloudMapFullAccess,arn:aws:iam::aws:policy/AWSAppMeshFullAccess \
    --override-existing-serviceaccounts \
    --approve
```

## App Mesh 컨트롤러를 배포합니다.
```
helm upgrade -i appmesh-controller eks/appmesh-controller \
    --namespace appmesh-system \
    --set region=ap-northeast-2 \
    --set serviceAccount.create=false \
    --set serviceAccount.name=appmesh-controller
```