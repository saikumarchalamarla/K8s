# 🚀 Kubernetes Monitoring with Prometheus: Node.js Page Hit Counter

## 📌 Overview
This project demonstrates how to:

- Build a Node.js application
- Instrument it with Prometheus metrics
- Containerize using Docker
- Deploy on Kubernetes (Minikube)
- Expose custom application metrics
- Configure Prometheus scraping
- Monitor application performance in real time

---

# 🧠 Architecture

```bash
User Traffic
    ↓
Node.js Application
    ↓
Prometheus Metrics Endpoint (/metrics)
    ↓
Kubernetes Service (Annotated)
    ↓
Prometheus Server
    ↓
Grafana / Prometheus UI
```

---

# 🧠 1. Application Development

## Objective:
Create a page hit counter that:
- Increments on each `/` request
- Exposes metrics via `/metrics`

---

## `app.js`
```javascript
const express = require('express');
const Prometheus = require('prom-client');

const app = express();
const port = process.env.PORT || 8000;

const counter = new Prometheus.Counter({
  name: 'http_page_hit',
  help: 'Total number of page hits'
});

app.get('/', (req, res) => {
  counter.inc();
  res.send(`Page hit count : ${counter.hashMap[''].value}`);
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', Prometheus.register.contentType);
  res.end(await Prometheus.register.metrics());
});

app.listen(port, () => {
  console.log(`App listening on port ${port}`);
});
```

---

# 🧠 2. Package Configuration

## `package.json`
```json
{
  "name": "http-page-hit",
  "version": "1.0.0",
  "main": "app.js",
  "dependencies": {
    "express": "^4.18.2",
    "prom-client": "^15.1.0"
  }
}
```

---

# 🧠 3. Dockerization

## Dockerfile
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

## Build Docker Image
```bash
docker build -t http-page-hit:v1 .
```

---

# 🧠 4. Load Image into Minikube

```bash
minikube image load http-page-hit:v1
```

---

# 🧠 5. Kubernetes Deployment

## `deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: http-page-hit
  namespace: office
spec:
  replicas: 2
  selector:
    matchLabels:
      app: http-page-hit
  template:
    metadata:
      labels:
        app: http-page-hit
    spec:
      containers:
      - name: http-page-hit
        image: http-page-hit:v1
        imagePullPolicy: Never
        ports:
        - containerPort: 8000
```

---

## Deploy:
```bash
kubectl apply -f deployment.yaml
```

---

# 🧠 6. Kubernetes Service with Prometheus Annotation

## `service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: http-page-hit
  namespace: office
  annotations:
    prometheus.io/scrape: "true"
spec:
  selector:
    app: http-page-hit
  type: NodePort
  ports:
    - port: 8000
      targetPort: 8000
      nodePort: 30800
```

---

## Deploy:
```bash
kubectl apply -f service.yaml
```

---

# 🧠 7. Validate Application

## Access externally:
```bash
minikube service http-page-hit -n office
```

---

## Internal test:
```bash
kubectl run testpod --image=busybox -it --rm -n office -- sh
wget -qO- http://http-page-hit:8000
```

### Example Output:
```bash
Page hit count : 1
Page hit count : 2
Page hit count : 3
```

---

# 🧠 8. Validate Metrics Endpoint

```bash
wget -qO- http://http-page-hit:8000/metrics
```

### Example Output:
```bash
# HELP http_page_hit Total number of page hits
# TYPE http_page_hit counter
http_page_hit 3
```

---

# 🧠 9. Prometheus Installation

## Add Helm Repo:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

## Install Prometheus:
```bash
helm install prom-demo prometheus-community/prometheus
```

---

## Expose Prometheus:
```bash
kubectl expose service prom-demo-server --type=NodePort --target-port=9090 --name=prometheus-service
```

---

## Access:
```bash
minikube service prometheus-service
```

---

# 🧠 10. Query Custom Metrics

## Prometheus Console Query:
```bash
http_page_hit
```

### Result:
Displays live page-hit counter metrics.

---

# 🧠 11. Networking Flow

```bash
Browser/User
   ↓
NodePort Service
   ↓
Node.js Pod
   ↓
/metrics endpoint
   ↓
Prometheus scraper
   ↓
Prometheus database
   ↓
Visualization / Alerting
```

---

# 🧠 12. Core Kubernetes Commands Used

## Pods:
```bash
kubectl get pods -n office
```

## Services:
```bash
kubectl get svc -n office
```

## Logs:
```bash
kubectl logs deployment/http-page-hit -n office
```

## Metrics:
```bash
kubectl exec -it deployment/http-page-hit -n office -- wget -qO- localhost:8000/metrics
```

---

# 🧠 13. Key Learning Outcomes

### Skills gained:
- Application instrumentation
- Prometheus metrics exposure
- Docker containerization
- Kubernetes deployment
- Service annotations
- Prometheus scraping
- Internal service networking
- Observability fundamentals

---

# 🧠 14. Common Troubleshooting

## Metrics not visible:
### Causes:
- `/metrics` endpoint missing
- Service annotation absent
- Prometheus config issue

---

## App inaccessible:
### Causes:
- Wrong NodePort
- Service selector mismatch
- Pod crash

---

## Debug:
```bash
kubectl describe svc http-page-hit -n office
kubectl get endpoints -n office
kubectl logs deployment/http-page-hit -n office
```

---

# 🧠 15. Enterprise Enhancements

### Future improvements:
- Grafana dashboards
- Alertmanager
- TLS security
- Prometheus Operator
- ServiceMonitor CRDs
- Horizontal scaling
- Distributed tracing
- CI/CD integration

---

# 📌 Resume-Grade Achievement

### Practical statement:
**“Developed and deployed a Prometheus-instrumented Node.js application on Kubernetes, enabling custom metrics exposure, service discovery, and real-time observability.”**

---

# ⭐ Conclusion

## Complete observability workflow:
```bash
Code
 ↓
Prometheus instrumentation
 ↓
Docker containerization
 ↓
Kubernetes deployment
 ↓
Service exposure
 ↓
Prometheus scraping
 ↓
Monitoring + Analytics
```

---

# 🚀 Final Takeaway

This project demonstrates real-world DevOps capability in:

- Monitoring
- Metrics engineering
- Kubernetes operations
- Production observability foundations

