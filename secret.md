```
apiVersion: v1
kind: Secret
metadata:
  name: db-config
type: Opaque
data:
  username: your_base64_encoded_username
  password: your_base64_encoded_password
  database: your_base64_encoded_database_name
  host: your_base64_encoded_endpoint_url
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp-container
        image: your_docker_image
        env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-config
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-config
              key: password
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: db-config
              key: database
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: db-config
              key: host
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: web-container
        image: 749692678017.dkr.ecr.ap-northeast-2.amazonaws.com/web
        ports:
        - containerPort: 3000
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
        env:
        - name: RDS_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-config
              key: username
        - name: RDS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-config
              key: password
        - name: RDS_HOSTNAME
          valueFrom:
            secretKeyRef:
              name: db-config
              key: host
      nodeSelector:
        app: web

```
