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

## 이후 단계에서 사용할 수 있도록 다음 변수를 설정합니다.
```
export CLUSTER_NAME=클러스터이름
export AWS_REGION=리전
```

## 클러스터에 대한 OpenID Connect(OIDC) 자격 증명 공급자를 만듭니다.
```
eksctl utils associate-iam-oidc-provider \
    --region=$AWS_REGION \
    --cluster $CLUSTER_NAME \
    --approve
```

## IAM 역할을 생성하고 여기에 AWSCloudMapFullAccessAWS관리 정책을 연결하고appmesh-controller Kubernetes 서비스 계정에 바인딩합니다. 
```
eksctl create iamserviceaccount \
    --cluster $CLUSTER_NAME \
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
    --set region=$AWS_REGION \
    --set serviceAccount.create=false \
    --set serviceAccount.name=appmesh-controller
```

# 매시 생성
app-mesh-contoller 설정이 완료되면 메시 생성을 진행할 수 있습니다.

### 네임스페이스를 생성합니다.
```
apiVersion: v1
kind: Namespace
metadata:
  name: my-apps
  labels:
    mesh: my-mesh
    appmesh.k8s.aws/sidecarInjectorWebhook: enabled
```
```
kubectl apply -f namespace.yaml
```

### App Mesh 서비스 메시를 생성합니다.
```
apiVersion: appmesh.k8s.aws/v1beta2
kind: Mesh
metadata:
  name: my-mesh
spec:
  namespaceSelector:
    matchLabels:
      mesh: my-mesh
```
```
kubectl apply -f mesh.yaml
```

### virtual-node.yml
```
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualNode
metadata:
  name: my-service-a
  namespace: my-apps
spec:
  podSelector:
    matchLabels:
      app: my-app-1
  listeners:
    - portMapping:
        port: 80
        protocol: http
  serviceDiscovery:
    dns:
      hostname: my-service-a.my-apps.svc.cluster.local
```
```
kubectl apply -f virtual-node.yaml
```
### virtual-route.yml
```
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualRouter
metadata:
  namespace: my-apps
  name: my-service-a-virtual-router
spec:
  listeners:
    - portMapping:
        port: 80
        protocol: http
  routes:
    - name: my-service-a-route
      httpRoute:
        match:
          prefix: /
        action:
          weightedTargets:
            - virtualNodeRef:
                name: my-service-a
              weight: 1
```
```
kubectl apply -f virtual-route.yml
```
## 컨트롤러가 App Mesh에서 생성한 가상 라우터 리소스를 확인합니다. 컨트롤러가 App Mesh에서 가상 라우터를 만들 때 가상 라우터 이름에 Kubernetes 네임스페이스 이름을 추가했기 때문에my-service-a-virtual-router_my-apps forname 를 지정합니다.
```
aws appmesh describe-virtual-router --virtual-router-name my-service-a-virtual-router_my-apps --mesh-name my-mesh
```
### virtual-service.yml
```
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualService
metadata:
  name: my-service-a
  namespace: my-apps
spec:
  awsName: my-service-a.my-apps.svc.cluster.local
  provider:
    virtualRouter:
      virtualRouterRef:
        name: my-service-a-virtual-router
```
```
kubectl apply -f virtual-service.yml
```

### App Mesh와 함께 사용하려는 모든 포드에는 App Mesh 사이드카 컨테이너가 추가되어 있어야 합니다. 인젝터는 사용자가 지정한 레이블에 배포된 모든 포드에 사이드카 컨테이너를 자동으로 추가합니다.
다음 콘텐츠를 컴퓨터에 proxy-auth.json이라는 파일에 저장합니다. 대체 색상 값을 고유한 값으로 바꿔야 합니다.
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "appmesh:StreamAggregatedResources",
            "Resource": [
                "arn:aws:appmesh:Region-code:111122223333:mesh/my-mesh/virtualNode/my-service-a_my-apps"
            ]
        }
    ]
}
```
### 정책을 생성합니다.
```
aws iam create-policy --policy-name my-policy --policy-document file://proxy-auth.json
```
### IAM 역할을 생성하고, 이전 단계에서 생성한 정책을 여기에 연결하고, Kubernetes 서비스 계정을 생성하고, 정책을 Kubernetes 서비스 계정에 바인딩합니다. 이 역할을 사용하면 컨트롤러가 App Mesh 리소스를 추가, 제거 및 변경할 수 있습니다.
```
eksctl create iamserviceaccount \
    --cluster $CLUSTER_NAME \
    --namespace my-apps \
    --name my-service-a \
    --attach-policy-arn  arn:aws:iam::111122223333:policy/my-policy \
    --override-existing-serviceaccounts \
    --approve
```
### Kubernetes 서비스 및 배포를 만듭니다. App Mesh와 함께 사용하려는 기존 배포가 있는 경우 하위 단계에서3 했던 것처럼 가상 노드를 배포해야2단계: App Mesh 리소스 배포 합니다. 배포를 업데이트하여 레이블이 가상 노드에 설정한 레이블과 일치하는지 확인하세요. 그러면 사이드카 컨테이너가 자동으로 포드에 추가되고 포드가 재배포됩니다.
```
apiVersion: v1
kind: Service
metadata:
  name: my-service-a
  namespace: my-apps
  labels:
    app: my-app-1
spec:
  selector:
    app: my-app-1
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service-a
  namespace: my-apps
  labels:
    app: my-app-1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app-1
  template:
    metadata:
      labels:
        app: my-app-1
    spec:
      serviceAccountName: my-service-a
      containers:
      - name: nginx
        image: nginx:1.19.0
        ports:
        - containerPort: 80
```
```
kubectl apply -f deployment.yml
```
