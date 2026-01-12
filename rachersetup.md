

# 🐮 Rancher Installation on kubeadm (No Domain, HTTPS via sslip.io)

This document explains a **complete, real-world Rancher installation** on a **kubeadm Kubernetes cluster** using:

* **NGINX Ingress Controller**
* **cert-manager + Let’s Encrypt**
* **sslip.io (no domain required)**
* **Bare EC2 / VM (no managed LoadBalancer)**

.

---

## 🧠 Architecture Overview

```
Internet
  ↓
Public IP (k8s-master)
  ↓
Ingress-NGINX (hostNetwork=true)
  ↓
Rancher Ingress
  ↓
Rancher Service
  ↓
Rancher Pod
```

TLS Flow:

```
Let’s Encrypt
  ↓
cert-manager
  ↓
tls-rancher-ingress (Secret)
  ↓
Ingress-NGINX
```

---

## 🧱 Cluster Assumptions

* Kubernetes installed via **kubeadm**
* 1 master, ≥1 worker
* Public IP attached to **master**
* No managed cloud LoadBalancer
* Helm v3 installed

---

## 1️⃣ Install Helm (Binary Method – Recommended)

```bash
curl -fsSL https://get.helm.sh/helm-v3.14.4-linux-amd64.tar.gz -o helm.tar.gz
tar -zxvf helm.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm version
```

---

## 2️⃣ Install cert-manager (Required for TLS)

### Add Helm repo

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

### Install cert-manager with CRDs

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

### Verify

```bash
kubectl get pods -n cert-manager
```

All pods must be `Running`.

---

## 3️⃣ Install NGINX Ingress Controller (Bare Metal / EC2)

### Add repo

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

### Install ingress-nginx

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.hostNetwork=true
```

---

## 4️⃣ Allow Ingress on Master Node (CRITICAL)

### Why this is needed

* Public IP is on **master**
* Master is tainted (`NoSchedule`)
* Ingress **must** run where the public IP exists

---

### Add tolerations + nodeSelector

```bash
kubectl patch deployment ingress-nginx-controller \
  -n ingress-nginx \
  --type=json \
  -p='[
    {
      "op": "add",
      "path": "/spec/template/spec/nodeSelector",
      "value": {
        "kubernetes.io/hostname": "k8s-master"
      }
    },
    {
      "op": "add",
      "path": "/spec/template/spec/tolerations",
      "value": [
        {
          "key": "node-role.kubernetes.io/control-plane",
          "operator": "Exists",
          "effect": "NoSchedule"
        },
        {
          "key": "node-role.kubernetes.io/master",
          "operator": "Exists",
          "effect": "NoSchedule"
        }
      ]
    }
  ]'
```

### Verify ingress runs on master

```bash
kubectl get pods -n ingress-nginx -o wide
```

Expected:

```
NODE = k8s-master
```

---

## 5️⃣ Verify Port 80 Is Listening (MANDATORY)

Run **on master**:

```bash
ss -lntp | grep ':80'
```

Expected:

```
LISTEN ... nginx
```

---

## 6️⃣ Install Rancher Using sslip.io

### Get public IP

```bash
curl ifconfig.me
```

Assume:

```
<PUBLIC_IP>
```

### Rancher hostname

```
rancher.<PUBLIC_IP>.sslip.io
```

---

### Add Rancher Helm repo

```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
```

### Create namespace

```bash
kubectl create namespace cattle-system
```

### Install Rancher

```bash
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.<PUBLIC_IP>.sslip.io \
  --set replicas=1 \
  --set ingress.tls.source=letsEncrypt
```

---

## 7️⃣ Fix IngressClass Mismatch (MOST COMMON ISSUE)

### Problem

* Ingress exists
* TLS exists
* Browser shows `404 nginx`
* IngressClass is `<none>`

### Verify

```bash
kubectl describe ingress rancher -n cattle-system
```

Look for:

```
Ingress Class: <none>
```

---

### Fix (MANDATORY)

```bash
kubectl patch ingress rancher -n cattle-system --type=merge -p '{
  "spec": {
    "ingressClassName": "nginx"
  }
}'
```

---

## 8️⃣ Force Fresh TLS Issuance (cert-manager Recovery)

If certificate gets stuck in `False`:

```bash
kubectl delete certificate tls-rancher-ingress -n cattle-system
```

cert-manager will auto-recreate it.

Verify:

```bash
kubectl get certificates -n cattle-system -w
```

Expected:

```
READY = True
```

---

## 9️⃣ Final Verification Checklist

```bash
kubectl get certificates -n cattle-system
# READY = True

kubectl get ingress -n cattle-system
# CLASS = nginx

kubectl get pods -n ingress-nginx
# Running on master
```

---

## 🔐 Access Rancher UI

```
https://rancher.<PUBLIC_IP>.sslip.io
```

### First-time login URL

```bash
echo https://rancher.<PUBLIC_IP>.sslip.io/dashboard/?setup=$(kubectl get secret -n cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')
```

---

## 🧠 Key Lessons (Production Reality)

* **Ingress ≠ Ingress Controller**
* **IngressClass must match everywhere**
* **cert-manager does not fix old ACME objects**
* **Master taints block scheduling**
* **Public IP must align with ingress node**
* **Deleting a Certificate is safe and often required**

---

