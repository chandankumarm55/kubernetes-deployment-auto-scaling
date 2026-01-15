# 🚀 Portfolio Project — Docker → Kubernetes → Scaling Guide

This README explains **how to deploy the Portfolio React app using Docker and Kubernetes**, and how to **scale it and test load** step by step.

> ✅ Tested with: Windows + Docker Desktop + Minikube (kubectl)

---

## 📌 Prerequisites

Make sure you have:

* Docker Desktop installed
* Kubernetes enabled (Docker Desktop) **or** Minikube running
* kubectl installed
* Docker Hub account

Check:

```bash
kubectl version --client
kubectl config current-context
```

---

## 🧱 Step 1 — Dockerize the React App (Nginx)

### 📄 Dockerfile

```dockerfile
# Step 1: Build React app
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Step 2: Serve using Nginx
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

> ⚠️ Note: If using Create-React-App, replace `/app/dist` with `/app/build`.

---

## 🐳 Step 2 — Build & Push Docker Image

```bash
docker build -t chandankumar55/portfolio:latest .
docker login
docker push chandankumar55/portfolio:latest
```

**Why?**
Kubernetes pulls images from registries like Docker Hub, not from your local system.

---

## ☸️ Step 3 — Kubernetes Deployment

### 📄 deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portfolio-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: portfolio
  template:
    metadata:
      labels:
        app: portfolio
    spec:
      containers:
      - name: portfolio
        image: chandankumar55/portfolio:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
```

### 📄 service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: portfolio-service
spec:
  type: NodePort
  selector:
    app: portfolio
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30006
```

Apply:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Check:

```bash
kubectl get pods
kubectl get svc
```

Access:

```
http://localhost:30006
```

---

## 🔁 Step 4 — Manual Scaling (Horizontal Scaling)

Increase pods:

```bash
kubectl scale deployment portfolio-deployment --replicas=3
```

Verify:

```bash
kubectl get pods
```

### ✅ How traffic is handled

Kubernetes Service load-balances requests:

```
Request → Pod A
Request → Pod B
Request → Pod C
```

---

## 📈 Step 5 — Auto Scaling with HPA (Horizontal Pod Autoscaler)

> ⚠️ HPA requires **Metrics Server**. Works best in **Minikube** or cloud clusters.

### Enable metrics in Minikube

```bash
minikube addons enable metrics-server
```

Verify:

```bash
kubectl top nodes
kubectl top pods
```

---

### 📄 hpa.yaml

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: portfolio-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: portfolio-deployment
  minReplicas: 1
  maxReplicas: 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Apply:

```bash
kubectl apply -f hpa.yaml
```

Check:

```bash
kubectl get hpa
```

---

## 🔥 Step 6 — Generate Load (Test Autoscaling)

### Delete old test pods

```bash
kubectl delete pod load-test
```

### Create BusyBox load pod

```bash
kubectl run load-test --image=busybox --restart=Never -it -- sh
```

Inside pod:

```sh
while true; do wget -q -O- http://portfolio-service; done
```
![Kubernetes Auto Scaling Output](https://raw.githubusercontent.com/<your-username>/<repo-name>/main/images/scaling.png)

### Watch scaling

In another terminal:

```bash
kubectl get pods -w
```

If CPU increases and metrics are working, new pods will be created automatically.

---

## ⚠️ Important Reality

Frontend apps (React + Nginx):

* Very low CPU usage
* HPA may NOT trigger easily

Best autoscaling demo:

* Backend APIs (Node / Java / Python)

---

## 🔄 Rolling Updates (Zero Downtime)

Push new image version and update:

```bash
kubectl set image deployment/portfolio-deployment portfolio=chandankumar55/portfolio:latest
```

Kubernetes will:

* Create new pod
* Remove old pod
* No downtime

---


---

## 📦 Production Architecture

```
Users
  ↓
Ingress (Nginx)
  ↓
Service
  ↓
Multiple Pods (Auto-scaled)
  ↓
Backend APIs
  ↓
Database (Replica Set)
```


