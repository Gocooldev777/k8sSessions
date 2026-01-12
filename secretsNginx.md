# 🔐 NGINX with Basic Authentication (Using Kubernetes Secrets)

**Namespace:** `ert`  
**Auth type:** HTTP Basic Auth  
**Image:** `nginx:1.25-alpine` (public, stable)

This is a **production-correct, clean NGINX Basic Auth setup using Kubernetes Secrets**. No private images. No registry auth. No surprises.

---

## 1️⃣ Create Namespace (if not already)

```bash
kubectl create namespace ert
```

---

## 2️⃣ Create htpasswd File (Auth Credentials)

Run this on your node or local machine:

```bash
apk add apache2-utils   # alpine
# OR
apt install apache2-utils -y   # ubuntu
```

Create credentials:

```bash
htpasswd -c auth admin
```

You'll be prompted for a password.

This creates a file named `auth`.

---

## 3️⃣ Create Kubernetes Secret from htpasswd

```bash
kubectl create secret generic nginx-basic-auth \
  --from-file=auth \
  -n ert
```

Verify:

```bash
kubectl get secret nginx-basic-auth -n ert
```

---

## 4️⃣ NGINX Config (Basic Auth Enabled)

```yaml
# nginx-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: ert
data:
  default.conf: |
    server {
        listen 80;

        location / {
            auth_basic "Restricted Area";
            auth_basic_user_file /etc/nginx/auth/auth;

            root /usr/share/nginx/html;
            index index.html;
        }
    }
```

Apply:

```bash
kubectl apply -f nginx-config.yaml
```

---

## 5️⃣ NGINX Deployment (Auth + Secret Mounted)

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: ert
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
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
        - name: auth-volume
          mountPath: /etc/nginx/auth
          readOnly: true
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
      - name: auth-volume
        secret:
          secretName: nginx-basic-auth
```

Apply:

```bash
kubectl apply -f nginx-deployment.yaml
```

---

## 6️⃣ Expose NGINX via Service

```yaml
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: ert
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

Apply:

```bash
kubectl apply -f nginx-service.yaml
```

---

## 7️⃣ Test from Inside the Cluster

### Create test pod

```bash
kubectl run test-client \
  --image=curlimages/curl \
  -n ert \
  -- sleep 3600
```

### ❌ Without credentials

```bash
kubectl exec -n ert test-client -- curl http://nginx
```

Expected:

```
401 Authorization Required
```

### ✅ With credentials

```bash
kubectl exec -n ert test-client -- \
curl -u admin:<password> http://nginx
```

Expected:

```
<!DOCTYPE html>
<html>
...
```

---

## 8️⃣ Why This Is the **RIGHT WAY**

✔ Secrets store sensitive data  
✔ Secrets mounted as **files**, not env vars  
✔ NGINX uses standard `htpasswd`  
✔ No private images  
✔ Production-grade pattern

---

## 9️⃣ Explanation

> **Kubernetes Secrets securely store credentials, which are mounted into pods and consumed by applications like NGINX without hardcoding sensitive data.**

---




# 🔐 Kubernetes Secrets – Types & When to Use What

**Kubernetes Secrets** are used to store **sensitive data** such as passwords, tokens, keys, and certificates, separately from application code and configuration.

---

## 🧠 Big Picture (Before Types)

```
Secret  →  Pod  →  Container
           |
           ├── Environment Variable
           └── Mounted File
```

Secrets are:

* Stored in **etcd**
* Base64-encoded (not encrypted by default)
* Protected via **RBAC**
* Namespaced (except service account tokens)

---

## 1️⃣ Opaque Secret (Most Common)

### 🔹 Type

```yaml
type: Opaque
```

### 🔹 What It Is

A **generic key–value secret**.

### 🔹 Used For

* Database username/password
* API keys
* Tokens
* Application secrets

### 🔹 Example

```bash
kubectl create secret generic app-secret \
  --from-literal=DB_PASSWORD=secret123 \
  -n ert
```

### 🔹 When to Use

✅ Default choice  
✅ 80% of real-world use cases  
❌ Not specialized (manual handling)

---

## 2️⃣ docker-registry Secret (Image Pull Secret)

### 🔹 Type

```yaml
type: kubernetes.io/dockerconfigjson
```

### 🔹 What It Is

Stores **container registry credentials**.

### 🔹 Used For

* Pulling images from:
  * Private Docker Hub
  * GHCR
  * ECR
  * GCR
  * ACR

### 🔹 Example

```bash
kubectl create secret docker-registry regcred \
  --docker-server=ghcr.io \
  --docker-username=user \
  --docker-password=token \
  -n ert
```

### 🔹 When to Use

✅ Private container images  
❌ Not for application secrets

---

## 3️⃣ TLS Secret (Certificates)

### 🔹 Type

```yaml
type: kubernetes.io/tls
```

### 🔹 What It Is

Stores **TLS certificates and private keys**.

### 🔹 Used For

* HTTPS (Ingress)
* mTLS
* Secure internal communication

### 🔹 Required Keys

```yaml
tls.crt
tls.key
```

### 🔹 Example

```bash
kubectl create secret tls tls-secret \
  --cert=cert.pem \
  --key=key.pem \
  -n ert
```

### 🔹 When to Use

✅ Ingress TLS  
✅ Secure services  
❌ Not for passwords or tokens

---

## 4️⃣ Service Account Token Secret

### 🔹 Type

```yaml
type: kubernetes.io/service-account-token
```

### 🔹 What It Is

Automatically generated token for **Kubernetes API access**.

### 🔹 Used For

* Pods talking to Kubernetes API
* Controllers
* Operators

### 🔹 Important

⚠️ **Auto-mounted into pods by default**

### 🔹 When to Use

✅ Internal cluster API access  
❌ Never store manually  
❌ Do not expose externally

---

## 5️⃣ Basic Auth Secret

### 🔹 Type

```yaml
type: kubernetes.io/basic-auth
```

### 🔹 What It Is

Stores:

* `username`
* `password`

### 🔹 Used For

* Simple HTTP auth
* Legacy systems

### 🔹 Example

```yaml
data:
  username: YWRtaW4=
  password: cGFzcw==
```

### 🔹 When to Use

✅ Simple authentication demos  
❌ Not recommended for modern apps

---

## 6️⃣ SSH Auth Secret

### 🔹 Type

```yaml
type: kubernetes.io/ssh-auth
```

### 🔹 What It Is

Stores SSH private keys.

### 🔹 Used For

* Git access
* CI/CD pipelines

### 🔹 Key

```yaml
ssh-privatekey
```

### 🔹 When to Use

✅ GitOps tools  
❌ Avoid mounting broadly

---

## 7️⃣ Bootstrap Token Secret (Advanced)

### 🔹 Used Internally By Kubernetes

* Node joining
* Cluster bootstrapping

### 🔹 When to Use

❌ Almost never manually  
✅ Managed by cluster admins

---

## 📊 Quick Decision Table

| Use Case               | Secret Type          |
| ---------------------- | -------------------- |
| App password / API key | Opaque               |
| Private image pull     | docker-registry      |
| HTTPS / Ingress TLS    | TLS                  |
| Pod → K8s API          | ServiceAccount token |
| Simple HTTP auth       | basic-auth           |
| Git / SSH access       | ssh-auth             |

---

## 🛑 What NOT to Do

❌ Put secrets in ConfigMaps  
❌ Commit secrets to Git  
❌ Share secrets across namespaces  
❌ Rely only on Base64 for security

---

## 🔐 Production Best Practices

* Enable **etcd encryption**
* Use **RBAC** strictly
* Prefer **file-mounted secrets**
* Rotate secrets periodically
* Use **external secret managers** for production

---

## 🌍 When Kubernetes Secrets Are NOT Enough

Use external managers when you need:

* Automatic rotation
* Centralized secrets
* Audit logs
* Cross-cluster sharing

Examples:

* AWS Secrets Manager
* HashiCorp Vault
* Azure Key Vault

---

## ✅ Final One-Liner

> **Kubernetes Secrets securely store sensitive data, while different secret types exist to match specific use cases such as app credentials, registry access, TLS, and API authentication.**

---

--------------------------------------------------------------------------------------------







---

# 🔐 NGINX with Application Credentials (Using Kubernetes App Secrets)

**Namespace:** `ert`
**Auth type:** App-level Username & Password (via headers / env / file)
**Secret type:** `Opaque`
**Image:** `nginx:1.25-alpine`

This demonstrates **how applications consume secrets**, not HTTP Basic Auth.

---

## 1️⃣ Create Application Credentials Secret

This time, we **DO NOT use htpasswd**.

```bash
kubectl create secret generic app-secret \
  --from-literal=APP_USERNAME=appuser \
  --from-literal=APP_PASSWORD=apppassword \
  -n ert
```

Verify:

```bash
kubectl get secret app-secret -n ert
```

---

## 2️⃣ NGINX Config (Uses App Secrets)

We’ll make NGINX **expect credentials via headers**.

```yaml
# nginx-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: ert
data:
  default.conf: |
    server {
        listen 80;

        location / {
            if ($http_x_app_user != "appuser") {
                return 401;
            }
            if ($http_x_app_pass != "apppassword") {
                return 401;
            }

            root /usr/share/nginx/html;
            index index.html;
        }
    }
```

⚠️ This is **only for demo clarity** — credentials will be injected, not hardcoded.

---

## 3️⃣ Inject App Secrets into NGINX Pod (ENV → NGINX)

### Deployment

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app-auth
  namespace: ert
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-app-auth
  template:
    metadata:
      labels:
        app: nginx-app-auth
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        env:
        - name: APP_USERNAME
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: APP_USERNAME
        - name: APP_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: APP_PASSWORD
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
```

Apply:

```bash
kubectl apply -f nginx-config.yaml
kubectl apply -f nginx-deployment.yaml
```

---

## 4️⃣ Expose NGINX via Service

```yaml
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-app-auth
  namespace: ert
spec:
  selector:
    app: nginx-app-auth
  ports:
  - port: 80
    targetPort: 80
```

Apply:

```bash
kubectl apply -f nginx-service.yaml
```

---

## 5️⃣ Test from Inside the Cluster

### Test pod

```bash
kubectl run test-client \
  --image=curlimages/curl \
  -n ert \
  -- sleep 3600
```

---

### ❌ Without App Secrets

```bash
kubectl exec -n ert test-client -- \
curl http://nginx-app-auth
```

Expected:

```
401 Unauthorized
```

---

### ❌ Wrong Credentials

```bash
kubectl exec -n ert test-client -- \
curl -H "X-App-User: wrong" -H "X-App-Pass: wrong" http://nginx-app-auth
```

Expected:

```
401 Unauthorized
```

---

### ✅ Correct App Credentials

```bash
kubectl exec -n ert test-client -- \
curl -H "X-App-User: appuser" -H "X-App-Pass: apppassword" http://nginx-app-auth
```

Expected:

```
<!DOCTYPE html>
<html>
...
```

---

## 6️⃣ What Changed from Basic Auth?

| Basic Auth           | App Secret         |
| -------------------- | ------------------ |
| Browser-supported    | App-controlled     |
| htpasswd file        | Opaque Secret      |
| Authorization header | Custom headers     |
| User-facing          | Service-to-service |

---

## 7️⃣ Why This Pattern Matters

✔ App secrets are **not tied to HTTP**
✔ Works for:

* Frontend → Backend
* Backend → DB
* Worker → API

✔ Secrets can be:

* Rotated
* Externalized (AWS)
* Scoped per namespace

---

## 8️⃣ Explanation

> **Application secrets allow services to authenticate with each other using credentials injected securely into pods, without hardcoding values in code or manifests.**

---

## 9️⃣ Production Notes (Important)

❌ Don’t hardcode values in NGINX config (done here for clarity)
✅ Real apps read secrets dynamically
✅ Prefer **file-mounted secrets** for rotation
✅ Move to **External Secrets** for AWS

---

-------------------
