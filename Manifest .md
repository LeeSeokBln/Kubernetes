Deployment
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <>
  namespace: <>
spec:
  selector:
    matchLabels:
      type: <>
  replicas: <>
  template:
    metadata:
      labels:
        type: <>
    spec:
      containers:
      - name: <>
        image: <>
        ports:
        - containerPort: <>
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
```

Service
```
apiVersion: v1
kind: Service

metadata:
  name: <>
  namespace: <>
spec:
  selector:
    type: stress
  ports:
  - port: <>
    targetPort: <>
    protocol: TCP
  type: NodePort
```
Ingress
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: stress-ingress
  namespace: skills
  annotations:
    alb.ingress.kubernetes.io/load-balancer-name: <>
    alb.ingress.kubernetes.io/scheme: <>
    alb.ingress.kubernetes.io/subnets: <>
    alb.ingress.kubernetes.io/target-type: <>
    alb.ingress.kubernetes.io/healthcheck-path: <>
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: <>
            port:
              number: <>
```