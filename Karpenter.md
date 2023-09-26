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
```
export KARPENTER_VERSION=v0.30.0
export AWS_PARTITION="aws" # if you are not using standard partitions, you may need to configure to aws-cn / aws-us-gov
export CLUSTER_NAME="<cluster_name>"
export AWS_DEFAULT_REGION="ap-northeast-2"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export TEMPOUT=$(mktemp)
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
curl -fsSL https://raw.githubusercontent.com/aws/karpenter/"${KARPENTER_VERSION}"/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml  > $TEMPOUT \
&& aws cloudformation deploy \
  --stack-name "Karpenter-${CLUSTER_NAME}" \
  --template-file "${TEMPOUT}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${CLUSTER_NAME}"
```
```
karpenter:
  version: 'v0.30.0'
  createServiceAccount: true
iam:
  withOIDC: true
  serviceAccounts:
  - metadata:
      name: karpenter
      namespace: karpenter
    roleName: <cluster_name>-karpenter
    attachPolicyARNs:
    - arn:aws:iam::749692678017:policy/KarpenterControllerPolicy-<cluster_name>
    roleOnly: true
```
추가
```
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version v0.30.0 --namespace karpenter --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::749692678017:role/skills-cluster-karpenter \
  --set settings.aws.clusterName=skills-cluster \
  --set settings.aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-skills-cluster \
  --set settings.aws.interruptionQueueName=skills-cluster \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait
```
CPU사용률 확인
```
watch kubectl get hpa -n <ns>
```

테스트
```
stress -c 100
```
