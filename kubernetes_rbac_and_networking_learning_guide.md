# 🚀 Kubernetes RBAC, Networking & Enterprise Security Guide

## 📌 Overview
This guide covers **hands-on Kubernetes RBAC setup, pod-to-pod communication, service exposure, and enterprise-grade security architecture** using Minikube.

Focus is on:
- Authentication
- Authorization
- Role-Based Access Control (RBAC)
- Pod communication
- Services
- UI exposure
- Enterprise security practices

---

# 🧠 1. Minikube RBAC Cluster Setup

## Start Minikube with RBAC:
```bash
minikube start --driver=docker --extra-config=apiserver.authorization-mode=RBAC
```

### Purpose:
Enables Kubernetes authorization controls.

---

# 🧠 2. Create Employee User Certificates

## Generate private key:
```bash
openssl genrsa -out employee.key 2048
```

## Generate CSR:
```bash
openssl req -new -key employee.key -out employee.csr -subj "/CN=employee/O=fresco"
```

### Meaning:
- CN = Username
- O = Group

---

# 🧠 3. Sign User Certificate

```bash
openssl x509 -req -in employee.csr -CA ~/.minikube/ca.crt -CAkey ~/.minikube/ca.key -CAcreateserial -out employee.crt -days 500
```

---

# 🧠 4. Create Namespace

```bash
kubectl create namespace office
```

---

# 🧠 5. Add Credentials + Context

## Set credentials:
```bash
kubectl config set-credentials employee --client-certificate=employee.crt --client-key=employee.key
```

## Set context:
```bash
kubectl config set-context employee-context --cluster=minikube --namespace=office --user=employee
```

---

# 🧠 6. Initial Access Test

```bash
kubectl --context=employee-context get pods
```

### Expected:
Forbidden

### Why:
Authentication successful, authorization missing.

---

# 🧠 7. Create Role

## ops-role.yaml
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: office
  name: ops-role
rules:
- apiGroups: ["", "apps", "extensions"]
  resources:
    - pods
    - pods/log
    - pods/attach
    - pods/exec
    - deployments
    - services
  verbs:
    - get
    - list
    - watch
    - create
    - delete
```

## Apply:
```bash
kubectl apply -f ops-role.yaml
```

---

# 🧠 8. Create RoleBinding

## ops-binding.yaml
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ops-binding
  namespace: office
subjects:
- kind: User
  name: employee
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: ops-role
  apiGroup: rbac.authorization.k8s.io
```

## Apply:
```bash
kubectl apply -f ops-binding.yaml
```

---

# 🧠 9. Test RBAC Permissions

## Allowed:
```bash
kubectl --context=employee-context get pods
kubectl --context=employee-context run ui-pod --image=busybox -it --rm -- sh
```

---

# 🧠 10. Pod-to-Pod Communication

## Expose NGINX deployment:
```bash
kubectl expose deployment nginx --port=80 --target-port=80 --name=nginx-service
```

---

## From UI pod:
```bash
wget -qO- http://nginx-service
```

### Result:
UI pod successfully accesses nginx.

---

# 🧠 11. Why Services Matter

### Without Service:
- Pod IP changes
- No stable DNS
- Fragile architecture

### With Service:
- Stable DNS
- Internal load balancing
- Reliable communication

---

# 🧠 12. Expose UI to Browser

## Create UI deployment:
```bash
kubectl create deployment ui --image=nginx -n office
```

## Expose:
```bash
kubectl expose deployment ui --type=NodePort --port=80 -n office
```

## Access:
```bash
minikube service ui -n office
```

---

# 🧠 13. Beginner Architecture

```bash
Browser
   ↓
NodePort Service
   ↓
UI Deployment
   ↓
UI Pod
   ↓
nginx-service
   ↓
NGINX Pod
```

---

# 🧠 14. Enterprise Security Evolution

## Required layers:

### Namespace Isolation
- frontend
- backend
- database

---

### RBAC
- Developers
- Operators
- Security teams

---

### Network Policies
Restrict pod communication.

Example:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ui-to-nginx
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: ui
```

---

### Ingress Controller
- HTTPS
- TLS termination
- Domain routing

---

### Secrets Management
- Kubernetes Secrets
- Hashicorp Vault
- Azure Key Vault

---

### Service Accounts
Avoid default identities.

---

# 🧠 15. Enterprise Architecture

```bash
Internet
   ↓
Ingress + TLS + WAF
   ↓
Frontend Service
   ↓
Backend Service
   ↓
Application Layer
   ↓
Database Layer
```

---

# 🧠 16. Security Comparison

| Level | Beginner | Enterprise |
|------|----------|------------|
| Access | NodePort | Ingress + WAF |
| Security | Basic RBAC | Granular RBAC |
| Networking | Flat | NetworkPolicy |
| Secrets | Hardcoded | Vault |
| Encryption | HTTP | HTTPS/TLS |
| Identity | Default SA | Dedicated SA |
| Monitoring | Minimal | Prometheus/Grafana |

---

# 🧠 17. Real Production Tools

### Networking:
- NGINX Ingress
- Istio
- Linkerd

### Security:
- OPA Gatekeeper
- Falco
- Azure AD Workload Identity

### Monitoring:
- Prometheus
- Grafana
- Loki

---

# 🧠 18. Common Errors Faced

## VirtualBox failure:
```bash
HOST_VIRT_UNAVAILABLE
```
### Fix:
Use Docker driver.

---

## Driver mismatch:
```bash
GUEST_DRIVER_MISMATCH
```
### Fix:
```bash
minikube delete
```

---

## Docker daemon issue:
```bash
PROVIDER_DOCKER_VERSION_EXIT_1
```
### Fix:
Start Docker Desktop.

---

## Forbidden RBAC errors:
### Cause:
Missing verbs/resources.

### Fix:
Expand role permissions.

---

# 📌 Conclusion

This setup teaches:

## Beginner:
- Minikube
- RBAC basics
- Pod communication
- Services
- NodePort

## Intermediate:
- Role separation
- Service architecture
- Deployment exposure

## Enterprise:
- Zero Trust networking
- TLS
- Vault
- Ingress
- NetworkPolicy
- Production security

---

# ⭐ Key Real-World Takeaway

## Kubernetes maturity path:
```bash
Pods → Services → Deployments → RBAC → NetworkPolicy → Ingress → Enterprise Security
```

Mastering this progression builds actual production Kubernetes engineering capability.

