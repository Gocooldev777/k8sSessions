# 🧠 Kubernetes Networking – Hands-On Lab

**Pod-to-Pod Networking · NetworkPolicy · MetalLB**

---

## 0️⃣ Pre-checks (MANDATORY)

Run this on **master node**:

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

✅ You should see:
* All nodes **Ready**
* Core pods running (`kube-system`)

---

## 1️⃣ Pod-to-Pod Networking (Core Concept)

### 🔹 What we are proving

* Pods can talk to each other **across nodes**
* Kubernetes networking model guarantees:
  * **Every pod has a unique IP**
  * **No NAT between pods**
  * **Pod-to-Pod communication works by default**

---

### 1.1 Create Test Namespace

```bash
kubectl create ns net-test
```

---

### 1.2 Create Two Pods (on different nodes)

📄 **pod-a.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-a
  namespace: net-test
  labels:
    app: a
spec:
  containers:
  - name: nginx
    image: nginx
```

📄 **pod-b.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-b
  namespace: net-test
  labels:
    app: b
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
```

Apply:

```bash
kubectl apply -f pod-a.yaml
kubectl apply -f pod-b.yaml
```

---

### 1.3 Verify Pod IPs

```bash
kubectl get pods -n net-test -o wide
```

You'll see:

```
pod-a   10.244.1.5   worker-1
pod-b   10.244.2.7   worker-2
```

👉 **Different nodes, different IP ranges**

---

### 1.4 Test Pod-to-Pod Connectivity

```bash
kubectl exec -n net-test pod-b -- ping -c 3 <POD-A-IP>
```

✅ **Expected Result**

```
64 bytes from 10.244.x.x: icmp_seq=1 ttl=64 time=0.x ms
```

🎯 **Conclusion**
* ✔ Pod-to-Pod networking is working
* ✔ CNI (Flannel / Calico) is doing its job

---

## 2️⃣ NetworkPolicy (OPTIONAL but IMPORTANT)

⚠ **VERY IMPORTANT RULE**

> NetworkPolicy works **ONLY if your CNI supports it**

* ❌ Flannel → **NO**
* ✅ Calico / Cilium → **YES**

Check your CNI:

```bash
kubectl get pods -n kube-system
```

If you see **calico-node**, you're good.

---

### 2.1 Default Behavior (No Policy)

By default:
* **ALL pods can talk to ALL pods**

We will now **BLOCK everything** and **allow selectively**.

---

### 2.2 Deny All Traffic (Baseline)

📄 **deny-all.yaml**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: net-test
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

Apply:

```bash
kubectl apply -f deny-all.yaml
```

---

### 2.3 Test Again (Should FAIL)

```bash
kubectl exec -n net-test pod-b -- ping -c 3 <POD-A-IP>
```

❌ **Expected**

```
ping: permission denied / timeout
```

🎯 **Now networking is locked**

---

### 2.4 Allow Traffic Only from pod-b → pod-a

📄 **allow-b-to-a.yaml**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-b-to-a
  namespace: net-test
spec:
  podSelector:
    matchLabels:
      app: a
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: b
```

Apply:

```bash
kubectl apply -f allow-b-to-a.yaml
```

---

### 2.5 Test Again (Should WORK)

```bash
kubectl exec -n net-test pod-b -- ping -c 3 <POD-A-IP>
```

✅ Success

🎯 **What students learn**
* Zero-trust networking
* East-West traffic control
* Real Kubernetes security model

---

## 3️⃣ MetalLB (LoadBalancer for Bare-Metal)

### 🔹 Why MetalLB?

* kubeadm / EC2 / on-prem **DO NOT support LoadBalancer**
* MetalLB gives **real external IPs**

---

### 3.1 Install MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```

Wait:

```bash
kubectl get pods -n metallb-system
```

✅ All pods should be **Running**

---

### 3.2 Choose IP Address Pool

Find your node IP range:

```bash
ip a
```

Example:

```
192.168.1.0/24
```

⚠ **Pick unused IPs ONLY**

---

### 3.3 Configure IP Pool

📄 **metallb-pool.yaml**

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2adv
  namespace: metallb-system
```

Apply:

```bash
kubectl apply -f metallb-pool.yaml
```

---

### 3.4 Create LoadBalancer Service

📄 **nginx-lb.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: net-test
spec:
  replicas: 2
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
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
  namespace: net-test
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

Apply:

```bash
kubectl apply -f nginx-lb.yaml
```

---

### 3.5 Verify External IP

```bash
kubectl get svc -n net-test
```

✅ You will see:

```
nginx-lb   LoadBalancer   10.x.x.x   192.168.1.240
```

---

### 3.6 Test from Browser / Curl

```bash
curl http://192.168.1.240
```

🎉 **NGINX Welcome Page appears**

---


---

**🏆 Congratulations! You've mastered Kubernetes networking fundamentals.**
