

## 1. Why Secrets Exist

In real-world applications, services need sensitive data such as:
- Usernames and passwords
- API keys
- Tokens
- Certificates

Storing these directly inside application code or Kubernetes YAML files is unsafe.


## 2. Application Flow (Mental Model)

```
User
  ↓
Frontend Pod
  ↓ (HTTP Request)
Backend Pod
  ↓
Business Logic / Database
```

Secrets are consumed by Pods but stored securely by Kubernetes.

---

## 3. Basic Setup: Frontend → Backend (No Security)

### 3.1 Create Namespace

```bash
kubectl create namespace app
```

---

### 3.2 Backend Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: app
spec:
  replicas: 1
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
        image: hashicorp/http-echo
        args:
        - "-text=Hello from Backend"
        ports:
        - containerPort: 5678
```

---

### 3.3 Backend Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: app
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 5678
```

Apply:

```bash
kubectl apply -f backend.yaml
kubectl apply -f backend-service.yaml
```

---

### 3.4 Frontend Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  namespace: app
spec:
  containers:
  - name: frontend
    image: curlimages/curl
    command: ["sleep", "3600"]
```

Apply:

```bash
kubectl apply -f frontend.yaml
```

Test connectivity:

```bash
kubectl exec -it frontend -n app -- curl backend-service
```

---

## 4. Introducing Credentials (Without Secrets)

Hardcoding credentials directly into Pods works but is unsafe.

```yaml
env:
- name: USERNAME
  value: admin
- name: PASSWORD
  value: admin123
```

This approach is discouraged and only shown to highlight the problem.

---

## 5. Kubernetes Secrets (Key-Value)

### 5.1 Create a Secret

```bash
kubectl create secret generic login-secret \
  --from-literal=username=admin \
  --from-literal=password=admin123 \
  -n app
```

Verify:

```bash
kubectl describe secret login-secret -n app
```

---

### 5.2 Consume Secret in Backend Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: app
spec:
  replicas: 1
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
        image: hashicorp/http-echo
        args:
        - "-text=Authenticated Backend"
        env:
        - name: USERNAME
          valueFrom:
            secretKeyRef:
              name: login-secret
              key: username
        - name: PASSWORD
          valueFrom:
            secretKeyRef:
              name: login-secret
              key: password
        ports:
        - containerPort: 5678
```

Apply:

```bash
kubectl apply -f backend-with-secret.yaml
```

Verify inside Pod:

```bash
kubectl exec -it <backend-pod> -n app -- env | grep USERNAME
```

---

## 6. File-Based Secrets (Real-World Usage)

Used for:

* TLS certificates
* SSH keys
* Nginx authentication
* API credential files

### 6.1 Create a File

```bash
echo "admin:password" > creds.txt
```

---

### 6.2 Create Secret from File

```bash
kubectl create secret generic file-secret \
  --from-file=creds.txt \
  -n app
```

---

### 6.3 Mount Secret as Volume

```yaml
volumeMounts:
- name: secret-volume
  mountPath: /etc/secrets
  readOnly: true

volumes:
- name: secret-volume
  secret:
    secretName: file-secret
```

Inside the container:

```
/etc/secrets/creds.txt
```

---

## 7. How Kubernetes Actually Protects Secrets

| Layer         | Responsibility       |
| ------------- | -------------------- |
| Secret Object | Stores encoded data  |
| Namespace     | Logical isolation    |
| RBAC          | Who can read secrets |
| Pod           | Consumes secrets     |
| NetworkPolicy | Who can talk to whom |

> Secrets are Base64-encoded, not encrypted by default.

---

## 8. Common Mistakes

* Committing secrets to Git
* Using ConfigMaps for passwords
* Using `--from-literal` incorrectly
* Mounting secrets to wrong paths
* Name mismatches between volume and volumeMount

---

## 9. Summary

* Secrets separate sensitive data from application logic
* Pods consume secrets, not define them
* Use literals for small values
* Use files for certs and configs
* Combine with RBAC and NetworkPolicy for real security
