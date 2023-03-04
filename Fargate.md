# EKS로 Fargate를 만들기 위한 클러스터 구성

### 기본 구성
```
fargateProfiles:
  - name: <>
    selectors:
      - namespace: <>
  - name: <>
    selectors:
      - namespace: <>
```
### 라벨이 필요할 때
```
selectors:
    labels:
        env: <>
        checks: <>

```