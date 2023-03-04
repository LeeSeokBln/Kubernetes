# EKS

## Install [eksctl]
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```
클러스터를 생성하기 위해 eksctl을 설치해야 합니다.

## Install [kubectl]
```
curl -o kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.23.7/2022-06-29/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
```
Node와 Pod를 관리하기 위해 kubectl을 설치해야 합니다.

---
[eksctl]: https://eksctl.io
[kubectl]: https://kubernetes.io

## Create Cluster.yaml
```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: <>
  region: ap-northeast-2
  version: "1.22"
cloudWatch:
  clusterLogging:
    enableTypes: ["*"]
iam:
  withOIDC: true
vpc:
  id:
  subnets:
    public:
      ap-northeast-2a-pub: { id: $subnet_id }
      ap-northeast-2b-pub: { id: $subnet_id }
      ap-northeast-2c-pub: { id: $subnet_id }
    private:
      ap-northeast-2a-priv: { id: $subnet_id }
      ap-northeast-2b-priv: { id: $subnet_id }
      ap-northeast-2c-priv: { id: $subnet_id }
managedNodeGroups:
  - name: <>
    instanceName: <>
    instanceType: <>
    minSize: <>
    maxSize: <>
    desiredCapacity: <>
    privateNetworking: true
    subnets:
     - ap-northeast-2a-priv
     - ap-northeast-2b-priv
     - ap-northeast-2c-priv
    iam:
      withAddonPolicies:
        imageBuilder: true
        autoScaler: true
        awsLoadBalancerController: true
        cloudWatch: true
```
