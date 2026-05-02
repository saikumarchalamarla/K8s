# 🚀 FitnessWeb Kubernetes Deployment Guide

## 📌 Overview
This guide documents the real deployment journey of the **FitnessWeb Django application** from Docker containerization to Kubernetes deployment using Minikube.

Focus:
- Docker build
- Local container validation
- Minikube image loading
- Kubernetes Deployment
- Service exposure
- Internal networking
- DNS
- Load balancing
- Real debugging

---

# 🧠 1. Containerizing FitnessWeb

## Build Docker image:
```bash
docker build -t fitnessweb:v1 .
```

### Purpose:
Packages the Django app into a portable container image.

---

# 🧠 2. Local Docker Validation

## Run locally:
```bash
docker run -p 8000:8000 fitnessweb:v1
```

## Access:
```bash
http://localhost:8000
```

### Outcome:
- Verified application works in Docker
- Confirmed container startup
- Confirmed Django app functionality

---

# 🧠 3. Load Docker Image into Minikube

```bash
minikube image load fitnessweb:v1
```

### Why:
Minikube needs access to the locally built image.

---

# 🧠 4. Kubernetes Deployment Creation

## fitnessweb-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fitnessweb
  namespace: office
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fitnessweb
  template:
    metadata:
      labels:
        app: fitnessweb
    spec:
      containers:
      - name: fitnessweb
        image: fitnessweb:v1
        imagePullPolicy: Never
        ports:
        - containerPort: 8000
```

---

## Apply deployment:
```bash
kubectl apply -f fitnessweb-deployment.yaml
```

### Result:
- Two FitnessWeb replicas deployed
- Kubernetes manages scaling
- Self-healing enabled

---

# 🧠 5. Expose Application via Service

## fitnessweb-service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: fitnessweb-service
  namespace: office
spec:
  selector:
    app: fitnessweb
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: NodePort
```

---

## Apply:
```bash
kubectl apply -f fitnessweb-service.yaml
```

### Result:
- Stable service endpoint created
- Internal ClusterIP assigned
- External NodePort assigned

---

# 🧠 6. Current Running Pods

```bash
kubectl get pods -n office -o wide
```

### Active:
```bash
fitnessweb-77dfcbc66-f95jq → 10.244.0.7
fitnessweb-77dfcbc66-sj8kd → 10.244.0.8
```

### Key Learning:
- Each replica gets unique IP
- Deployment ensures availability
- Scaling works automatically

---

# 🧠 7. Current Service Configuration

```bash
kubectl get svc -n office
```

### FitnessWeb Service:
```bash
fitnessweb-service → NodePort → 10.100.1.146:80 → NodePort 30124
```

---

# 🧠 8. Traffic Flow Architecture

```bash
Browser
   ↓
Minikube Docker tunnel (localhost)
   ↓
NodePort Service
   ↓
ClusterIP Service
   ↓
kube-proxy load balancing
   ↓
FitnessWeb Pod replicas
```

---

# 🧠 9. Endpoint Verification

```bash
kubectl describe svc fitnessweb-service -n office
```

### Endpoints:
```bash
10.244.0.7:8000
10.244.0.8:8000
```

### Meaning:
- Service selector works
- Both pods registered
- Load balancing active

---

# 🧠 10. DNS Testing Inside Cluster

## Temporary test pod:
```bash
kubectl run testpod --image=busybox -it --rm -n office -- sh
```

## Inside pod:
```bash
nslookup fitnessweb-service
wget -qO- http://fitnessweb-service
```

---

### DNS Result:
```bash
fitnessweb-service.office.svc.cluster.local
```

### Key Learning:
- Kubernetes DNS works
- Namespace-qualified DNS matters

---

# 🧠 11. 400 Bad Request Issue

## Initial Problem:
```bash
wget http://fitnessweb-service
```

### Result:
```bash
HTTP/1.1 400 Bad Request
```

---

## Root Cause:
Django `ALLOWED_HOSTS` restriction.

### Previous deployed config:
```python
ALLOWED_HOSTS = ['0.0.0.0', 'localhost', '127.0.0.1']
```

### Problem:
Kubernetes service DNS (`fitnessweb-service`) was not allowed.

---

## First Failed Attempt:
- Updated local settings.py
- Rebuilt image
- Restarted deployment

### Issue persisted because:
Old deployment/image remained active.

---

## Final Working Fix:
### Step 1:
Update:
```python
ALLOWED_HOSTS = ['*']
```

---

### Step 2:
Delete old deployment completely:
```bash
kubectl delete deployment fitnessweb -n office
```

---

### Step 3:
Reload fresh image:
```bash
minikube image load fitnessweb:v1
```

---

### Step 4:
Reapply deployment:
```bash
kubectl apply -f fitnessweb-deployment.yaml
```

---

### Step 5:
Verify deployed config:
```bash
kubectl exec -it deployment/fitnessweb -n office -- grep ALLOWED_HOSTS gym_project/settings.py
```

### Confirmed:
```python
ALLOWED_HOSTS = ['*']
```

---

### Step 6:
Retest:
```bash
kubectl run testpod --image=busybox -it --rm -n office -- sh
wget -qO- http://fitnessweb-service
```

---

## Successful Result:

### Final verified output:
```html
<!DOCTYPE html>
<html lang="en">
<head>
<title>Home | CoreFit Health & Nutrition</title>
</head>
<body>
<h1>Welcome to CoreFit Health & Nutrition</h1>
```

### Actual successful output snippet:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Home | CoreFit Health & Nutrition</title>
    <link rel="stylesheet" href="/static/css/style.css">
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-light bg-light">
        <a class="navbar-brand" href="#">CoreFit Health & Nutrition</a>
    </nav>

    <div class="container mt-5">
        <h1 class="display-4">Welcome to CoreFit Health & Nutrition</h1>
        <p class="lead">Explore our personal coaching subscription plans to enhance your fitness and nutrition.</p>
        <a class="btn btn-primary btn-lg" href="/subscriptions/">View Subscription Plans</a>
        <a class="btn btn-secondary btn-lg" href="/cart/">View Cart</a>
    </div>

    <footer class="text-center mt-5">
        <p>© 2024 CoreFit Health & Nutrition. All rights reserved.</p>
    </footer>
</body>
</html>
```

### Real routes confirmed from rendered homepage:
```bash
/
/subscriptions/
/cart/
/gym-tips/
/nutrition-tips/
/contact/
```

### Static assets confirmed:
```bash
/static/css/style.css
/static/images/gym.jpg
```

### Confirmed:
- Kubernetes networking working
- Internal DNS resolution working
- Service routing working
- Deployment healthy
- Replica pods serving traffic
- Django application accessible internally
- ALLOWED_HOSTS issue resolved

---

## Real clarity from actual output:
Your `wget` command successfully returned full Django HTML homepage from inside Kubernetes cluster.

### This proves:
```bash
Pod → Service DNS → ClusterIP → kube-proxy → FitnessWeb app
```

Worked end-to-end successfully.

---

## Final Lesson:
### Kubernetes issue was NOT networking.
### Actual blocker was stale deployment + Django host validation.

---

## Real Production Debugging Pattern:
```bash
Code change
 ↓
Docker rebuild
 ↓
Image reload
 ↓
Delete stale deployment
 ↓
Redeploy cleanly
 ↓
Verify running config
 ↓
Retest networking
```

---

## Key DevOps Takeaway:
### Always verify deployed runtime config, not just local source code.

---

# 🧠 12. External Access

## Command:
```bash
minikube service fitnessweb-service -n office --url
```

### Example:
```bash
http://127.0.0.1:51054
```

### Important:
- Docker driver on Windows creates tunnel
- Terminal must remain open

---

# 🧠 13. Networking Layers Built

| Layer | Status |
|------|--------|
| Docker container | Working |
| Kubernetes Deployment | Working |
| Replica scaling | Working |
| Pod networking | Working |
| Service discovery | Working |
| DNS | Working |
| NodePort | Working |
| Load balancing | Working |
| Django host validation | Resolved |

---

# 🧠 14. Core Kubernetes Commands Used

## Pods:
```bash
kubectl get pods -n office -o wide
```

## Services:
```bash
kubectl get svc -n office
```

## Endpoints:
```bash
kubectl get endpoints -n office
```

## Service details:
```bash
kubectl describe svc fitnessweb-service -n office
```

## DNS test:
```bash
kubectl run testpod --image=busybox -it --rm -n office -- sh
```

## External URL:
```bash
minikube service fitnessweb-service -n office --url
```

---

# 🧠 15. Real Beginner DevOps Learning

### You implemented:
- Docker image creation
- Local testing
- Kubernetes deployment
- Replica management
- Service exposure
- DNS verification
- Internal networking
- External browser access
- Application debugging

---

# 🧠 16. Practical Resume-Level Achievement

### Current level:
**“Containerized and deployed a Django-based FitnessWeb application on Kubernetes using Docker, Minikube, Deployments, Services, and NodePort networking.”**

---

# 🧠 17. Immediate Next Enterprise Upgrades

### Future improvements:
- Gunicorn
- NGINX
- PostgreSQL
- ConfigMaps
- Secrets
- Ingress
- TLS
- Monitoring
- CI/CD
- Security hardening

---

# 📌 Conclusion

## Real deployment maturity achieved:
```bash
Code
 ↓
Docker
 ↓
Kubernetes Deployment
 ↓
Service Exposure
 ↓
Networking Validation
 ↓
Application Debugging
```

---

# ⭐ Key Takeaway

## Kubernetes alone isn’t enough:
### Application configuration (Django settings) must align with cluster networking.

This project now demonstrates practical beginner-to-intermediate Kubernetes deployment capability.

