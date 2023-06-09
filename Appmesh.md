# AWS App Mesh는 서비스가 여러 유형의 컴퓨팅 인프라에서 서로 쉽게 통신할 수 있도록 애플리케이션 수준 네트워킹을 제공하는 서비스 메시입니다
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

# 매시 생성
app-mesh-contoller 설정이 완료되면 메시 생성을 진행할 수 있습니다.

### appmesh.yml
```
apiVersion: appmesh.k8s.aws/v1beta2
kind: Mesh
metadata:
  name: knol-mesh
spec:
  namespaceSelector:
    matchLabels:
      mesh: knol-mesh 
```
```
kubectl apply -f appmesh.yml
```

### 이제 메쉬가 생성되었으므로 아래와 같이 네임스페이스에 레이블을 추가하여 사이드카 주입을 활성화/비활성화할 수 있습니다.
```
apiVersion: v1
kind: Namespace
metadata:
  name: 네임스페이스
  labels:
    appmesh.k8s.aws/sidecarInjectorWebhook: enabled
```
```
kubectl apply -f namespace.yml
```

### virtual-node.yml
```
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualNode
metadata:
  name: knol-service
  namespace: 네임스페이스
spec:
  awsName: knol-service-virtual-node
  podSelector:
    matchLabels:
      app: knol-service
  listeners:
    - portMapping:
        port: 80
        protocol: http
  serviceDiscovery:
    dns:
      hostname: knol-service.mesh-workload.svc.cluster.local
```
```
kubectl apply -f virtual-node.yml
```
### virtual-route.yml
```
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualRouter
metadata:
  namespace: 네임스페이스
  name: knol-service
spec:
  awsName: knol-service-virtual-router
  listeners:
    - portMapping:
        port: 80
        protocol: http
  routes:
    - name: route
      httpRoute:
        match:
          prefix: /
        action:
          weightedTargets:
            - virtualNodeRef:
                name: knol-service
              weight: 1
```
```
kubectl apply -f virtual-route.yml
```
### virtual-service.yml
```
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualService
metadata:
  name: knol-service
  namespace: 네임스페이스
spec:
  awsName: knol-service-virtual-service
  provider:
    virtualRouter:
      virtualRouterRef:
        name: knol-service
```
```
kubectl apply -f virtual-service.yml
```

### serviceaccount.yml
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: knol-service
  namespace: 네임스페이스
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::Account_ID:role/appmesh_role
```
```
kubectl apply -f serviceaccount.yml
```
### deployment.yml
```
apiVersion: v1
kind: Service
metadata:
  name: knol-service
  namespace: 네임스페이스
  labels:
    app: knol-service
spec:
  selector:
    app: knol-service
  ports:
    - protocol: TCP
      port: 컨테이너 포트
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: knol-service
  namespace: 네임스페이스
  labels:
    app: knol-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: knol-service
  template:
    metadata:
      labels:
        app: knol-service
    spec:
      serviceAccountName: knol-service
      containers:
      - name: 컨테이너 이름
        image: 이미지
        ports:
          - containerPort: 포트
```
```
kubectl apply -f deployment.yml
```
