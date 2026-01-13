# Rancher Installation Guide - Bare Metal with kubeadm

This guide provides instructions for installing Rancher on a bare-metal Kubernetes cluster using kubeadm, with HTTPS via Let's Encrypt and sslip.io.

## Architecture Overview

- **Platform**: kubeadm-based Kubernetes cluster
- **Ingress**: NGINX Ingress Controller
- **Certificate Management**: cert-manager with Let's Encrypt
- **DNS**: sslip.io (no custom domain required)
- **Deployment Target**: Master node with hostNetwork

---

## Prerequisites

- Functional kubeadm cluster with master and worker nodes
- kubectl configured and accessible
- Helm 3.x installed
- Public IP address accessible on ports 80 and 443

---

## Part 1: Environment Cleanup

### Remove Existing Installations

```bash
helm uninstall rancher -n cattle-system || true
helm uninstall ingress-nginx -n ingress-nginx || true
helm uninstall cert-manager -n cert-manager || true
```

### Delete Namespaces

```bash
kubectl delete ns cattle-system ingress-nginx cert-manager --force --grace-period=0
```

Verify namespaces are removed:

```bash
kubectl get ns
```

### Remove Custom Resource Definitions

```bash
kubectl delete crd \
  certificates.cert-manager.io \
  certificaterequests.cert-manager.io \
  challenges.acme.cert-manager.io \
  clusterissuers.cert-manager.io \
  issuers.cert-manager.io \
  orders.acme.cert-manager.io || true
```

### Remove Webhook Configurations

```bash
kubectl delete validatingwebhookconfiguration cert-manager-webhook || true
kubectl delete mutatingwebhookconfiguration cert-manager-webhook || true
```

### Clean iptables (Master Node)

```bash
iptables -F
iptables -t nat -F
systemctl restart containerd kubelet
```

A node reboot is recommended after cleanup.

---

## Part 2: Cluster Verification

Verify all nodes are in Ready state:

```bash
kubectl get nodes -o wide
```

Verify only system pods are running:

```bash
kubectl get pods -A
```

---

## Part 3: Install Helm

```bash
curl -fsSL https://get.helm.sh/helm-v3.14.4-linux-amd64.tar.gz -o helm.tar.gz
tar -xzf helm.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm version
```

---

## Part 4: Install cert-manager

Add the Jetstack Helm repository:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

Install cert-manager with CRDs:

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

Verify all cert-manager pods are running:

```bash
kubectl get pods -n cert-manager
```

---

## Part 5: Install NGINX Ingress Controller

Add the NGINX Ingress Helm repository:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

### Install with Host Network Configuration

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.hostNetwork=true \
  --set controller.service.type=ClusterIP \
  --set controller.kind=Deployment
```

### Configure Master Node Scheduling

Apply node selector and tolerations to run ingress on the master node:

```bash
kubectl patch deployment ingress-nginx-controller \
  -n ingress-nginx \
  --type=json \
  -p='[
    {
      "op": "add",
      "path": "/spec/template/spec/nodeSelector",
      "value": {"node-role.kubernetes.io/control-plane": ""}
    },
    {
      "op": "add",
      "path": "/spec/template/spec/tolerations",
      "value": [{
        "key": "node-role.kubernetes.io/control-plane",
        "operator": "Exists",
        "effect": "NoSchedule"
      }]
    }
  ]'
```

### Verify Ingress Controller

Confirm the ingress controller pod is running on the master node:

```bash
kubectl get pods -n ingress-nginx -o wide
```

Verify ports 80 and 443 are bound:

```bash
ss -lntp | grep ':80'
ss -lntp | grep ':443'
```

---

## Part 6: Install Rancher

### Retrieve Public IP Address

```bash
PUBLIC_IP=$(curl -s ifconfig.me)
echo $PUBLIC_IP
```

### Add Rancher Helm Repository

```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
```

### Create Namespace

```bash
kubectl create namespace cattle-system
```

### Install Rancher

```bash
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.${PUBLIC_IP}.sslip.io \
  --set ingress.ingressClassName=nginx \
  --set ingress.tls.source=letsEncrypt \
  --set replicas=1
```

---

## Part 7: Monitor Certificate Provisioning

Watch the certificate provisioning process:

```bash
kubectl get challenges,orders,certificates -n cattle-system -w
```

The expected sequence is:
1. Challenge created
2. Order validated
3. Certificate status shows READY = True

If the certificate provisioning becomes stuck, delete and recreate the certificate:

```bash
kubectl delete certificate tls-rancher-ingress -n cattle-system
```

---

## Part 8: Verification

### Verify Ingress Configuration

```bash
kubectl get ingress -n cattle-system
```

Expected output should show `CLASS: nginx`

### Verify Certificate Status

```bash
kubectl get certificates -n cattle-system
```

Expected output should show `READY: True`

### Check Ingress Controller Logs

```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller | grep rancher
```

---

## Part 9: Access Rancher

### Access the Web Interface

Navigate to:
```
https://rancher.<PUBLIC_IP>.sslip.io
```

### Retrieve Bootstrap Password

Generate the initial login URL with bootstrap credentials:

```bash
echo https://rancher.${PUBLIC_IP}.sslip.io/dashboard/?setup=$(kubectl get secret -n cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')
```

---

## Troubleshooting

### Certificate Issues

If certificates fail to provision:
- Verify port 80 is accessible from the internet (required for Let's Encrypt HTTP-01 challenge)
- Check cert-manager logs: `kubectl logs -n cert-manager deploy/cert-manager`
- Verify ACME challenge status: `kubectl describe challenge -n cattle-system`

### Ingress Connectivity Issues

If Rancher is unreachable:
- Confirm ingress controller is running on the master node with the public IP
- Verify firewall rules allow traffic on ports 80 and 443
- Check ingress controller logs for errors

### Pod Scheduling Issues

If pods fail to schedule:
- Verify node taints and tolerations
- Check node resources: `kubectl describe nodes`
- Review pod events: `kubectl describe pod <pod-name> -n <namespace>`
