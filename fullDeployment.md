
# 🧠 Production Architecture (What We’re Building)

![Image](https://rajeshradhakrishnanmvk.github.io/images/2021-04-29-Securing_MicroFrontend_Deployment_in_Kuberentes/media/image1.png)

![Image](https://docs.nginx.com/nic/ic-high-level.png)

![Image](https://miro.medium.com/v2/resize%3Afit%3A1400/1%2AuDcoPBS-jsvDrIHEtnB7Hg.png)

```
Internet
   |
[ DNS ]
   |
[ Ingress Controller (Nginx) ]
   |
-------------------------------
|           |                |
Frontend    Backend        Auth / APIs
Service     Service
   |            |
Frontend Pods  Backend Pods
   |
-------------------------------
|   Databases / External APIs |
-------------------------------
```

---

# PHASE 1 — Namespace & Cluster Hygiene (MANDATORY)

### 1️⃣ Create isolated namespaces

```bash
kubectl create namespace frontend
kubectl create namespace backend
kubectl create namespace ingress
kubectl create namespace monitoring
```

👉 **Why**

* Security isolation
* Easier scaling
* Clean RBAC & observability
* Zero blast radius

---

### 2️⃣ Label worker nodes (optional but recommended)

```bash
kubectl label node worker-1 role=app
kubectl label node worker-2 role=app
```

---

# PHASE 2 — Ingress Controller (ENTRY POINT)

### 3️⃣ Install NGINX Ingress Controller (Production way)

![Image](https://docs.nginx.com/nic/ic-high-level.png)

![Image](https://i.sstatic.net/xY184.png)

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress \
  --set controller.replicaCount=2 \
  --set controller.nodeSelector.role=app
```

👉 **Why**

* Handles SSL, routing, load-balancing
* Required for domain-based routing
* Highly stable in production

---

# PHASE 3 — Backend (API Layer)

### 4️⃣ Backend Docker Image (Best Practice)

**Dockerfile**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json .
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

👉 Small image, fast startup, secure

---

### 5️⃣ Backend Deployment (Production-Ready)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: your-registry/backend:1.0.0
        ports:
        - containerPort: 3000
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
```

---

### 6️⃣ Backend Service (Internal Only)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: backend
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP
```

👉 **Never expose backend directly**

---

# PHASE 4 — Frontend (Static App Best Practice)

### 7️⃣ Frontend Dockerfile (NGINX Pattern)

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY . .
RUN npm install && npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

---

### 8️⃣ Frontend Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: your-registry/frontend:1.0.0
        ports:
        - containerPort: 80
```

---

### 9️⃣ Frontend Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: frontend
spec:
  selector:
    app: frontend
  ports:
  - port: 80
  type: ClusterIP
```

---

# PHASE 5 — Ingress Routing (MOST IMPORTANT)

### 🔟 Single Ingress for Entire App

![Image](https://miro.medium.com/1%2AzeDLEJFmTJJUckhzdBzx_w.gif)

![Image](https://miro.medium.com/v2/resize%3Afit%3A1200/1%2AFT6ZQau2LNT3JZ5d0YoIqQ.png)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fullstack-ingress
  namespace: ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: app.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-svc
            port:
              number: 80
```

👉 Frontend → `/`
👉 Backend → `/api`

---

# PHASE 6 — Secrets, Configs & Security (NON-NEGOTIABLE)

### 1️⃣1️⃣ Secrets

```bash
kubectl create secret generic backend-secrets \
  --from-literal=DB_PASSWORD=xxxx \
  -n backend
```

Mount via `envFrom`

---

### 1️⃣2️⃣ ConfigMaps

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: backend
data:
  NODE_ENV: "production"
```

---

### 1️⃣3️⃣ NetworkPolicy (Zero Trust)

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: backend-policy
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress
```

---

# PHASE 7 — Scaling & Reliability

### Horizontal Pod Autoscaler

```bash
kubectl autoscale deployment backend \
  --cpu-percent=70 \
  --min=3 \
  --max=10 \
  -n backend
```

---

# PHASE 8 — Observability (PRODUCTION MUST)

![Image](https://cdn.prod.website-files.com/681e366f54a6e3ce87159ca4/6877c683c0a47068f5e6609c_Blog-Kubernetes-Monitoring-with-Prometheus-4-Architecture-Overview.png)

![Image](https://miro.medium.com/v2/resize%3Afit%3A8342/1%2Av8gXcoKjDO_RqVJgWRLJYA.jpeg)

Install:

* **Prometheus + Grafana**
* **Loki / ELK for logs**

---

# PHASE 9 — CI/CD (NO MANUAL DEPLOYS)

**Pipeline Flow**

```
Git Push
 → Build Docker
 → Security Scan
 → Push to Registry
 → Helm Upgrade
```

Use:

* GitHub Actions / GitLab CI
* Helm charts (mandatory for prod)

---

# 🚀 FINAL PRODUCTION CHECKLIST

✅ Namespace isolation
✅ Ingress + SSL ready
✅ No NodePort exposed
✅ Resource limits set
✅ Probes enabled
✅ Secrets & ConfigMaps
✅ HPA enabled
✅ Monitoring & logging
✅ CI/CD automated
