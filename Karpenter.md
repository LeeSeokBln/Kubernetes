provisioner.yml
```
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
  namespace: <ns>
spec:
  requirements:
    - key: "karpenter.k8s.aws/instance-category"
      operator: In
      values: ["c", "m", "r"]
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["on-demand"]
  limits:
    resources:
      cpu: 1000
  providerRef:
    name: default
  consolidation: 
    enabled: true
---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default
  namespace: <ns>
spec:
  subnetSelector:
    karpenter.sh/discovery: <cluster_name>
  securityGroupSelector:
    karpenter.sh/discovery: <cluster_name>
```
hpa.yml
```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa
  namespace: <ns>
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: <deployment_name>
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 30
```
cluster.yml에
```
iamIdentityMappings:
- arn: "arn:aws:iam::749692678017:role/KarpenterNodeRole-<Cluster_name>"
  username: system:node:{{EC2PrivateDNSName}}
  groups:
  - system:bootstrappers
  - system:nodes
```
```
karpenter:
  version: 'v0.30.0'
  createServiceAccount: true
iam:
  withOIDC: true
  serviceAccounts:
  - metadata:
      name: aws-load-balancer-controller
      namespace: kube-system
    wellKnownPolicies:
      awsLoadBalancerController: true
  - metadata:
      name: karpenter
      namespace: karpenter
    roleName: <cluster_name>-karpenter
    attachPolicyARNs:
    - arn:aws:iam::749692678017:policy/KarpenterControllerPolicy-<cluster_name>
    roleOnly: true
```
추가 (ALB컨트롤러는 덤)

CPU사용률 확인
```
watch kubectl get hpa -n <ns>
```

테스트
```
stress -c 100
```
