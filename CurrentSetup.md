# 🚀 Kubernetes from Gokul Kumar Vishwanathan
### *Part of K8s training -> Cluster Setup*

---

<div align="center">

```ascii
    ⎈ KUBERNETES CLUSTER BUILD ⎈
    
    ╔═══════════════════════════════╗
    ║   REAL CLUSTER. REAL SKILLS.  ║
    ║   NO MINIKUBE. NO SHORTCUTS.  ║
    ╚═══════════════════════════════╝
```

**Master the Art of Container Orchestration**

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.29-blue?style=for-the-badge&logo=kubernetes)](https://kubernetes.io/)
[![Calico](https://img.shields.io/badge/Calico-v3.27-orange?style=for-the-badge&logo=networkmanager)](https://www.projectcalico.org/)
[![AWS](https://img.shields.io/badge/AWS-EC2-yellow?style=for-the-badge&logo=amazon-aws)](https://aws.amazon.com/)

</div>

---

## 🎯 What Have we done till npw

This is **production-grade infrastructure**:

```
┌─────────────────────────────────────────────┐
│           AWS VPC (CurrentSetup That we did) |
│  ┌───────────────────────────────────────┐  │
│  │   🎛️  MASTER NODE (Control Plane)     │  │
│  │   • API Server (Brain)                │  │
│  │   • etcd (Memory)                     │  │
│  │   • Scheduler (Task Manager)          │  │
│  └───────────────────────────────────────┘  │
│           ↓↓↓ Commands ↓↓↓                  │
│  ┌──────────────┐      ┌──────────────┐    │
│  │ 🔧 WORKER 1   │      │ 🔧 WORKER 2   │    │
│  │ Runs Pods    │      │ Runs Pods    │    │
│  │ Calico CNI   │      │ Calico CNI   │    │
│  └──────────────┘      └──────────────┘    │
└─────────────────────────────────────────────┘
```

---

## 🧠 The IP Address Architecture

**Most tutorials skip this. Don't be most tutorials.**

| IP Type | Source | Example | Purpose |
|---------|--------|---------|---------|
| **Node IPs** | AWS VPC | `172.31.5.42` | Physical machine addresses |
| **Pod IPs** | Calico CNI | `192.168.10.5` | Container addresses (cluster-only) |
| **Service IPs** | kube-proxy | `10.96.0.1` | Virtual load balancer IPs |

> **💡 A quick question**: Who is giving IPs to the PODs.

---

```

---

## 🔥 Phase 1: Node Preparation (All Nodes)

### 🏷️ Step 1: Give Your Nodes Identity

```bash
sudo hostnamectl set-hostname master  # or worker1, worker2
```

**🎓 Why This Matters**:  
Kubernetes tracks nodes by hostname.

---

### 💾 Step 2: Kill Swap (It's Mandatory)

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

**🎓 The Brutal Truth**:  
Kubernetes assumes memory is sacred. Swap = unpredictable performance = kubelet panic. No swap = happy kubelet = happy life.

---

### 🧬 Step 3: Kernel Module

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

**🎓 Translation for Humans**:
- `overlay`: Lets containers stack filesystems like LEGO blocks
- `br_netfilter`: Lets iptables inspect traffic between containers

---

### 🌐 Step 4: Networking

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

**🎓 Simple Example**:
- "Dear kernel, please forward packets between containers"
- "Also, let iptables see bridge traffic"
- "Make this permanent, please"

---

### 📦 Step 5: Install containerd

```bash
sudo apt update && sudo apt install -y containerd

# Configure with systemd cgroup driver
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

**🎓 Fundamental thing*:  
Kubernetes doesn't run containers. It's a control plane. containerd is the actual container runner. They must speak the same cgroup language (systemd) or chaos ensues.

---

### ⎈ Step 6: Install Kubernetes

```bash
# Setup Kubernetes package repo
sudo apt install -y apt-transport-https ca-certificates curl

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
| sudo gpg --dearmor -o /etc/apt/keyrings/k8s.gpg

echo "deb [signed-by=/etc/apt/keyrings/k8s.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" \
| sudo tee /etc/apt/sources.list.d/k8s.list


sudo apt update
sudo apt install -y kubelet kubeadm kubectl


sudo apt-mark hold kubelet kubeadm kubectl
```

**🎓 Meet Your Tools**:
- **kubeadm**: The cluster bootstrapper (one-time use)
- **kubelet**: The node agent (runs forever)
- **kubectl**: Your command-line weapon (use daily)

---

## 👑 Phase 2: Master Node Coronation

### 🎬 Step 7: Initialize the Cluster

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

**🎓 What Just Happened**:  
kubeadm just:
1. Generated certificates for secure communication
2. Started etcd (the cluster's brain database)
3. Launched API Server (the control center)
4. Created a Scheduler (the task dispatcher)
5. Reserved `192.168.0.0/16` for pod IPs

**⚠️ SAVE THE OUTPUT!** It contains the worker join command. Lose it = start over.

---

### 🔑 Step 8: Get Admin Access

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**🎓 Reality Check**:  
This file is your cluster's master key. It contains credentials to do *anything* in the cluster.

---

### 🕸️ Step 9: Deploy Calico

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

**🎓 Why CNI**:  
Without CNI, pods can't talk. Calico:
- Assigns IPs from `192.168.0.0/16`
- Creates routes between nodes
- Enables pod-to-pod communication across nodes

Watch the magic:
```bash
kubectl get pods -n kube-system -w
```

---

## 🤝 Phase 3: Worker Node

### 🔗 Step 10: Join the Cluster

**On each worker node**, run the command from `kubeadm init` output:

```bash
sudo kubeadm join <MASTER_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

**🎓 Actual Operation as i Said, joining the kubeadm command that we ran already**:
1. Worker authenticates with master using token
2. Establishes secure TLS connection
3. kubelet registers node with API server
4. Calico configures networking

---

## ✅ Validation: Proof It Works

### Check Node Status
```bash
kubectl get nodes
```

**Expected Output**:
```
NAME      STATUS   ROLES           AGE   VERSION
master    Ready    control-plane   5m    v1.29.0
worker1   Ready    <none>          2m    v1.29.0
worker2   Ready    <none>          2m    v1.29.0
```

---

### Launch Your First Pod
```bash
kubectl run create-pod --image=nginx --port=80
kubectl get pod -o wide
```

**🎓 What to Look For**:
- Pod gets IP from `192.168.x.x` range
- Status shows `Running`
- Pod is scheduled on a worker node

---

## 🎨 Visual Debugging Cheat Sheet

```
┌─────────────────────────────────────┐
│ Pod stuck in "Pending"?             │
│ → Check: kubectl get events         │
│ → Likely: CNI not installed         │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Node shows "NotReady"?              │
│ → Check: kubectl describe node      │
│ → Likely: kubelet crashed           │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Can't reach pod IP?                 │
│ → Remember: Pod IPs are internal    │
│ → Use: kubectl port-forward         │
└─────────────────────────────────────┘
```

---

## 🧪 Points

### 🔍 Master Debugging Commands

```bash
# Describe a Pod
kubectl describe pod <pod-name>

# Check the logs
kubectl logs <pod-name>

# SSH into a pod (mostly i screw up in this syntax)
kubectl exec -it <pod-name> -- /bin/bash

# Check cluster health
kubectl get componentstatuses

# See all resources in all namespaces
kubectl get all --all-namespaces
```

---

### 🚨 Common Pitfalls

| Problem | Symptom | Fix |
|---------|---------|-----|
| Forgot to disable swap | kubelet won't start | `swapoff -a` |
| Wrong cgroup driver | Pods crash repeatedly | Check containerd config |
| CNI not installed | Pods stuck in "Pending" | Install Calico |
| Token expired | Join fails | Generate new token |
| Security group closed | Nodes can't communicate | Open port 6443 |

---

## 🎓 Conceptual Deep Dive: The Data Flow

```
User: kubectl run nginx --image=nginx
  ↓
kubectl → API Server (validates request)
  ↓
Scheduler → "Which node has resources?"
  ↓
Kubelet (chosen node) → "Got it, pulling image"
  ↓
containerd → Downloads nginx image
  ↓
Calico CNI → "Here's your pod IP: 192.168.5.10"
  ↓
Pod Status → Running ✓
```

---


---

## 📚 Essential Resources

- [Official Kubernetes Docs](https://kubernetes.io/docs/) - The source
- [Calico Documentation](https://docs.tigera.io/calico/latest/about/) - CNI
- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) - Manual setup

---



<div align="center">



</div>
