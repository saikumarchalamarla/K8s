# 🚀 Kubernetes Two-Tier Application Deployment with Ingress

## 📌 Overview
This hands-on project demonstrates how to deploy a multi-application Kubernetes environment using:

- Minikube
- Docker
- Kubernetes Deployments
- Services
- Ingress
- Namespace isolation
- Node.js application
- MongoDB
- NGINX

The environment contains:

1. MongoDB pod
2. Node.js page-hit application pod
3. Independent NGINX pod
4. Ingress routing

---

# 🧠 Architecture

```bash
                Ingress
                   ↓
        ----------------------
        ↓                    ↓
     /nginx               /app
        ↓                    ↓
   nginx-service      node-app-service
        ↓                    ↓
    nginx pod         nodejs pod
                             ↓
                        mongo-service
                             ↓
                         mongo pod
```

---

# 🧠 1. Environment Setup

## Verify installations

```bash
docker --version
minikube version
kubectl version --client
```

---

# 🧠 2. Start Minikube

```bash
minikube start --driver=docker
```

---

# 🧠 3. Sync Docker with Minikube

## Windows PowerShell

```powershell
minikube docker-env --shell powershell | Invoke-Expression
```

---

# 🧠 4. Verify Cluster

```bash
kubectl get nodes
```

### Expected

```bash
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   XXm   v1.xx.x
```

---

# 🧠 5. Create Project Folder

```powershell
cd $HOME\Desktop
mkdir kubernetes-page-count
cd kubernetes-page-count
```

---

# 🧠 6. Create Node.js Application

## app.js

```javascript
const express = require('express');
const app = express();

let count = 0;

app.get('/', (req, res) => {
  count++;
  res.send(`page hit count is: ${count}`);
});

app.listen(8000, () => {
  console.log('App running on port 8000');
});
```

---

# 🧠 7. Create package.json

```json
{
  "name": "node-mongo-page-hit",
  "version": "1.0.0",
  "main": "app.js",
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

---

# 🧠 8. Dockerfile

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package.json .
RUN npm install

COPY . .

EXPOSE 8000

CMD ["node", "app.js"]
```

---

# 🧠 9. Build Docker Image

```bash
docker build -t fresco/node-mongo-page-hit:latest .
```

---

# 🧠 10. Verify Docker Image

```bash
docker images
```

### Expected

```bash
REPOSITORY                         TAG       IMAGE ID
fresco/node-mongo-page-hit         latest    xxxxxxxxx
```

---

# 🧠 11. Create Namespace

```bash
kubectl create namespace frescons
```

---

# 🧠 12. Create Deployments

## deployments.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo
  namespace: frescons
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - name: mongo
        image: mongo:4.1
        ports:
        - containerPort: 27017

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-mongo-page-hit
  namespace: frescons
spec:
  replicas: 1
  selector:
    matchLabels:
      app: node-mongo-page-hit
  template:
    metadata:
      labels:
        app: node-mongo-page-hit
    spec:
      containers:
      - name: node-mongo-page-hit
        image: fresco/node-mongo-page-hit:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 8000
        env:
        - name: PORT
          value: "8000"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: frescons
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - echo 'Hello from Fresco Team' > /usr/share/nginx/html/index.html
```

---

# 🧠 13. Apply Deployments

```bash
kubectl apply -f deployments.yaml
```

---

# 🧠 14. Verify Pods

```bash
kubectl get pods -n frescons
```

### Example Output

```bash
mongo-xxxxxxxxxx-xxxxx                 Running
node-mongo-page-hit-xxxxxxxxxx-xxxxx  Running
nginx-xxxxxxxxxx-xxxxx                Running
```

---

# 🧠 15. Create Services

## services.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo
  namespace: frescons
spec:
  selector:
    app: mongo
  ports:
  - port: 27017
    targetPort: 27017

---
apiVersion: v1
kind: Service
metadata:
  name: node-mongo-page-hit
  namespace: frescons
spec:
  selector:
    app: node-mongo-page-hit
  type: NodePort
  ports:
  - port: 8000
    targetPort: 8000
    nodePort: 30800

---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: frescons
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

---

# 🧠 16. Apply Services

```bash
kubectl apply -f services.yaml
```

---

# 🧠 17. Verify Services

```bash
kubectl get svc -n frescons
```

### Example Output

```bash
mongo                  ClusterIP
node-mongo-page-hit    NodePort
nginx                  NodePort
```

---

# 🧠 18. Enable Ingress Addon

```bash
minikube addons enable ingress
```

---

# 🧠 19. Verify Ingress Controller

```bash
kubectl get pods -n ingress-nginx
```

### Wait until:

```bash
STATUS = Running
```

---

# 🧠 20. Create Ingress

## ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fresco-ingress
  namespace: frescons
spec:
  rules:
  - http:
      paths:
      - path: /nginx
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80

      - path: /app
        pathType: Prefix
        backend:
          service:
            name: node-mongo-page-hit
            port:
              number: 8000
```

---

# 🧠 21. Apply Ingress

```bash
kubectl apply -f ingress.yaml
```

---

# 🧠 22. Verify Ingress

```bash
kubectl get ingress -n frescons
```

---

# 🧠 23. Start Minikube Tunnel

```bash
minikube tunnel
```

Keep terminal open.

---

# 🧠 24. Test NGINX Route

```bash
curl -kL http://localhost/nginx
```

### Expected Output

```bash
Hello from Fresco Team
```

---

# 🧠 25. Test Node.js Route

```bash
curl -kL http://localhost/app
```

### Expected Output

```bash
page hit count is: 1
```

Refresh again:

```bash
page hit count is: 2
```

---

# 🧠 26. Current Application State

## MongoDB
Currently deployed but not yet integrated into Node.js logic.

Current counter behavior:

```bash
In-memory counter
```

If pod restarts:

```bash
Counter resets
```

---

# 🧠 27. Kubernetes Concepts Practiced

### Completed:

- Deployments
- Services
- NodePort
- Namespace isolation
- Docker image building
- Ingress routing
- Lifecycle hooks
- Multi-application deployment
- Internal Kubernetes networking

---

# 🧠 28. Lifecycle Hook Demonstration

NGINX deployment uses:

```yaml
lifecycle:
  postStart:
```

This automatically writes:

```bash
Hello from Fresco Team
```

into:

```bash
/usr/share/nginx/html/index.html
```

when container starts.

---

# 🧠 29. Traffic Flow

```bash
User
 ↓
Ingress
 ↓
/nginx → nginx-service → nginx pod
/app → node-service → node pod
```

---

# 🧠 30. Debug Commands

## Pods

```bash
kubectl get pods -n frescons
```

---

## Services

```bash
kubectl get svc -n frescons
```

---

## Ingress

```bash
kubectl get ingress -n frescons
```

---

## Logs

```bash
kubectl logs deployment/nginx -n frescons
kubectl logs deployment/node-mongo-page-hit -n frescons
```

---

# 🧠 31. Enterprise Evolution Path

Future improvements:

- MongoDB integration
- Persistent Volumes
- ConfigMaps
- Secrets
- HTTPS/TLS
- Horizontal Pod Autoscaling
- Monitoring with Prometheus
- Grafana dashboards
- CI/CD pipelines

---

# 📌 Resume-Level Statement

### Practical achievement:

**“Built and deployed a multi-application Kubernetes environment using Deployments, Services, Ingress, Docker, and Minikube with traffic routing and lifecycle hook automation.”**

---

# ⭐ Conclusion

## Full deployment workflow:

```bash
Docker Build
 ↓
Kubernetes Deployments
 ↓
Services
 ↓
Ingress Routing
 ↓
Browser Access
```

---

# 🚀 Final Outcome

Successfully deployed:

- MongoDB pod
- Node.js page-hit application
- NGINX pod
- Ingress routing
- Namespace-isolated Kubernetes environment

