# 🔐 etcd Backup & Restore — Complete Hands-On Guide

## 🎯 The One-Line Truth

> **etcd backup = cluster backup**  
> **No etcd snapshot = no Kubernetes upgrade**

---

## 🗺️ Big Picture (Read Once, Then Steps Will Click)

During a Kubernetes upgrade:

- ✅ Your **entire cluster state** lives inside etcd:
  - Pods, Services, Deployments
  - Secrets, ConfigMaps
  - RBAC policies
  - Node information
- ✅ If etcd is safe → **cluster is recoverable**
- ❌ If etcd is lost → **cluster is gone**

**Therefore:** etcd backup = cluster backup

---

## 🧠 Upgrade-Safe Mental Model

```
┌─────────────────────────────────┐
│   Take etcd snapshot            │  ← MOST IMPORTANT
└───────────┬─────────────────────┘
            ↓
┌─────────────────────────────────┐
│   Upgrade control plane         │
│   (kubeadm upgrade apply)       │
└───────────┬─────────────────────┘
            ↓
┌─────────────────────────────────┐
│   Upgrade worker nodes          │
│   (kubeadm upgrade node)        │
└───────────┬─────────────────────┘
            ↓
     ┌──────┴──────┐
     │             │
  Success?      Failure?
     │             │
     ✅            ❌
                   ↓
        ┌──────────────────────┐
        │  RESTORE etcd        │
        │  (cluster rollback)  │
        └──────────────────────┘
```

---

## 📋 What You'll Learn

1. ✅ Verify etcd configuration
2. ✅ Take etcd snapshots (backups)
3. ✅ Verify backup integrity
4. ✅ Restore etcd from snapshot
5. ✅ Understand why it works
6. ✅ Avoid common mistakes

---

## 🧪 Part 1: Verify etcd Details (Before Backup)

### Step 1: Check etcd Pod

On the **control plane node**:

```bash
kubectl -n kube-system get pods | grep etcd
```

**Output:**
```
etcd-master-node   1/1   Running   0   10d
```

### Step 2: Inspect etcd Manifest

```bash
sudo cat /etc/kubernetes/manifests/etcd.yaml
```

### Step 3: Find Critical Paths

Look for these **4 critical values**:

```yaml
--cert-file=/etc/kubernetes/pki/etcd/server.crt
--key-file=/etc/kubernetes/pki/etcd/server.key
--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
--data-dir=/var/lib/etcd
```

> 💡 **Key Understanding:** These paths are **MANDATORY** for backup & restore operations.

### Step 4: Verify Certificate Files Exist

```bash
ls -l /etc/kubernetes/pki/etcd/
```

**Expected output:**
```
ca.crt
ca.key
server.crt
server.key
peer.crt
peer.key
```

---

## 🧪 Part 2: Take etcd Backup (SNAPSHOT)

### Step 1: Set etcdctl API Version

```bash
export ETCDCTL_API=3
```

> 💡 **Why?** etcdctl has v2 and v3 APIs. Kubernetes uses v3.

### Step 2: Take Snapshot (THIS IS THE BACKUP)

Run on the **control plane node**:

```bash
sudo etcdctl snapshot save /opt/etcd-backup-$(date +%F).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**Expected output:**
```
Snapshot saved at /opt/etcd-backup-2026-01-03.db
```

### What Just Happened?

- ✅ etcdctl connected to etcd server on port 2379
- ✅ Used TLS certificates for authentication
- ✅ Created a **point-in-time snapshot** of entire cluster state
- ✅ Saved to `/opt/etcd-backup-2026-01-03.db`

### Step 3: Verify Snapshot (VERY IMPORTANT)

```bash
sudo etcdctl snapshot status /opt/etcd-backup-2026-01-03.db
```

**Expected output:**
```
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 12abc3d4 |    45678 |       1234 |      3.5MB |
+----------+----------+------------+------------+
```

### What This Tells You

- **REVISION:** etcd transaction ID at backup time
- **TOTAL KEYS:** Number of objects in cluster
- **TOTAL SIZE:** Backup file size

> 🔥 **If this command works → backup is VALID** ✅

### Step 4: Copy Backup OFF the Server (Production Rule)

```bash
# Copy to remote backup server
scp /opt/etcd-backup-2026-01-03.db user@backup-server:/backups/

# Or copy to your local machine
scp root@master-node:/opt/etcd-backup-2026-01-03.db ~/kubernetes-backups/
```

> ⚠️ **CRITICAL RULE:** Never keep the only backup on the same node. If the node dies, backup dies with it.

---

## 🧪 Part 3: Upgrade Kubernetes (High-Level Context)

### Normal Upgrade Commands

```bash
# Check available versions
sudo kubeadm upgrade plan

# Apply upgrade
sudo kubeadm upgrade apply v1.28.0

# Upgrade kubelet and kubectl
sudo apt-get update
sudo apt-get install -y kubelet=1.28.0-00 kubectl=1.28.0-00

# Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### If Anything Breaks

> 🚨 **DO NOT PANIC** → You restore etcd.

---

## 🧪 Part 4: Restore etcd (FULL CLUSTER ROLLBACK)

This restores your cluster to the **EXACT state** at backup time.

### 🚨 VERY IMPORTANT RULES Before Restore

1. ✅ **STOP kubelet** (stops all control plane components)
2. ✅ **MOVE** old etcd data (never delete directly)
3. ✅ **RESTORE** snapshot to same data directory
4. ✅ **START** kubelet again

---

### Step 1: Stop kubelet (on Control Plane)

```bash
sudo systemctl stop kubelet
```

**What this stops:**
- API Server
- etcd
- Scheduler
- Controller Manager

### Step 2: Move Existing etcd Data (Never Delete Directly)

```bash
sudo mv /var/lib/etcd /var/lib/etcd.broken.$(date +%s)
```

> 💡 **Why move instead of delete?** Safety. If restore fails, you can move it back.

**Verify it's moved:**
```bash
ls -ld /var/lib/etcd*
```

**Output:**
```
drwx------ 3 root root 4096 Jan  3 10:00 /var/lib/etcd.broken.1704268800
```

### Step 3: Restore Snapshot

```bash
sudo etcdctl snapshot restore /opt/etcd-backup-2026-01-03.db \
  --data-dir=/var/lib/etcd
```

**Expected output:**
```
2026-01-03 10:05:23.123456 I | mvcc: restore compact to 45678
2026-01-03 10:05:23.234567 I | etcdserver/membership: added member xyz [...]
Snapshot restored successfully
```

### What Just Happened?

- ✅ etcdctl read the snapshot file
- ✅ Recreated the etcd data directory
- ✅ Restored all cluster state to backup time

### Step 4: Fix Ownership (CRITICAL)

```bash
sudo chown -R root:root /var/lib/etcd
```

> ⚠️ **Why?** etcd runs as root and needs proper permissions.

### Step 5: Start kubelet Again

```bash
sudo systemctl start kubelet
```

**Wait 30–60 seconds** for all components to start.

### Step 6: Verify Cluster Restored

```bash
# Check nodes
kubectl get nodes

# Check pods
kubectl get pods -A

# Check deployments
kubectl get deployments -A

# Check services
kubectl get svc -A
```

### What You Should See

- ✅ Same nodes as before
- ✅ Same pods as before
- ✅ Same namespaces
- ✅ Same workloads
- ✅ Same configurations

> 🎯 **Cluster is rolled back successfully!** ✅

---

## 🧪 Part 5: Why This Works (Core Understanding)

### What Gets Stored in etcd?

| Component | Why It Survives Restore |
|-----------|-------------------------|
| **Pods** | Desired state stored in etcd |
| **Deployments** | Stored in etcd |
| **ReplicaSets** | Stored in etcd |
| **Services** | Stored in etcd |
| **Secrets** | Stored in etcd |
| **ConfigMaps** | Stored in etcd |
| **RBAC policies** | Stored in etcd |
| **Node info** | Stored in etcd |
| **PersistentVolumes** | Metadata in etcd |

### What About Worker Nodes?

Worker nodes:
- ✅ Reconnect automatically to API Server
- ✅ Reconcile their state from etcd
- ✅ Restart containers if needed

### The Complete Picture

```
┌─────────────────────────────────────────────────────┐
│                    etcd Database                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  All cluster state:                          │   │
│  │  - Pods, Services, Deployments              │   │
│  │  - Secrets, ConfigMaps                      │   │
│  │  - RBAC, Network Policies                   │   │
│  │  - Node registrations                       │   │
│  └─────────────────────────────────────────────┘   │
└────────────────────┬────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
    Snapshot                  Restore
        │                         │
        ↓                         ↓
  /opt/backup.db         Cluster rollback
   (point-in-time)       (exact state)
```

---

## ⚠️ Common Mistakes (Real-World Failures)

| Mistake | Why It Fails | Correct Approach |
|---------|--------------|------------------|
| ❌ Taking backup **after** upgrade | Backup has broken state | Take backup **before** upgrade |
| ❌ Forgetting cert flags | etcdctl can't authenticate | Always include `--cacert`, `--cert`, `--key` |
| ❌ Restoring without stopping kubelet | etcd and API Server conflict | Always stop kubelet first |
| ❌ **Deleting** `/var/lib/etcd` | Can't recover if restore fails | **Move** instead: `mv /var/lib/etcd /var/lib/etcd.old` |
| ❌ Not verifying snapshot | Unknown if backup is valid | Always run `snapshot status` |
| ❌ Keeping only one backup on same node | Node failure = backup loss | Copy to remote location |

---

## 🎓 Production Best Practices

### 1. Automated Backup Schedule

Create a cron job for daily backups:

```bash
# Edit crontab
sudo crontab -e

# Add daily backup at 2 AM
0 2 * * * ETCDCTL_API=3 /usr/local/bin/etcdctl snapshot save /backup/etcd-$(date +\%F).db --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key
```

### 2. Backup Retention Policy

```bash
# Keep last 7 days of backups
find /backup -name "etcd-*.db" -mtime +7 -delete
```

### 3. Off-Site Backup

```bash
# Upload to S3
aws s3 cp /opt/etcd-backup-$(date +%F).db s3://my-k8s-backups/

# Or use rsync
rsync -avz /opt/etcd-backup-*.db backup-server:/remote/backups/
```

### 4. Test Restore Regularly

```bash
# Monthly restore test on a test cluster
# Verify all objects come back
# Document restore time
```

---

## 📊 Complete Backup & Restore Workflow

```
┌─────────────────────────────────────────────────────┐
│          BEFORE ANY CLUSTER CHANGE                   │
└───────────────────┬─────────────────────────────────┘
                    ↓
         ┌──────────────────────┐
         │  1. Take Snapshot    │
         │  etcdctl snapshot    │
         │  save                │
         └──────────┬───────────┘
                    ↓
         ┌──────────────────────┐
         │  2. Verify Snapshot  │
         │  snapshot status     │
         └──────────┬───────────┘
                    ↓
         ┌──────────────────────┐
         │  3. Copy Off-Site    │
         │  scp/rsync/s3        │
         └──────────┬───────────┘
                    ↓
         ┌──────────────────────┐
         │  4. Perform Upgrade  │
         │  kubeadm upgrade     │
         └──────────┬───────────┘
                    ↓
              ┌─────┴─────┐
              │           │
          Success?    Failure?
              │           │
              ✅          ❌
                          ↓
              ┌───────────────────┐
              │  5. Stop kubelet  │
              └───────┬───────────┘
                      ↓
              ┌───────────────────┐
              │  6. Move old etcd │
              └───────┬───────────┘
                      ↓
              ┌───────────────────┐
              │  7. Restore       │
              │  snapshot         │
              └───────┬───────────┘
                      ↓
              ┌───────────────────┐
              │  8. Start kubelet │
              └───────┬───────────┘
                      ↓
              ┌───────────────────┐
              │  9. Verify        │
              │  cluster state    │
              └───────────────────┘
```

---

## 🔍 Verification Commands Summary

### Check etcd Health
```bash
# Check etcd pod
kubectl -n kube-system get pods | grep etcd

# Check etcd member list
sudo etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Check Backup
```bash
# List backups
ls -lh /opt/etcd-backup-*.db

# Check backup integrity
sudo etcdctl snapshot status /opt/etcd-backup-2026-01-03.db
```

### Check Cluster After Restore
```bash
# Check all resources
kubectl get all -A

# Check cluster info
kubectl cluster-info

# Check component status
kubectl get cs
```

---

## 🧹 Cleanup Old Backups

```bash
# Manual cleanup (keep last 7 days)
find /opt -name "etcd-backup-*.db" -mtime +7 -delete

# Or create cleanup script
cat > /usr/local/bin/cleanup-etcd-backups.sh <<'EOF'
#!/bin/bash
BACKUP_DIR="/opt"
RETENTION_DAYS=7
find $BACKUP_DIR -name "etcd-backup-*.db" -mtime +$RETENTION_DAYS -delete
echo "Cleaned up backups older than $RETENTION_DAYS days"
EOF

chmod +x /usr/local/bin/cleanup-etcd-backups.sh
```

---

## 🎯 One-Line Production Rule (Memorize This)

> **No etcd snapshot = no Kubernetes upgrade**

---

## 📝 Quick Reference Cheat Sheet

### Take Backup
```bash
export ETCDCTL_API=3
sudo etcdctl snapshot save /opt/backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Verify Backup
```bash
sudo etcdctl snapshot status /opt/backup.db
```

### Restore Backup
```bash
sudo systemctl stop kubelet
sudo mv /var/lib/etcd /var/lib/etcd.old
sudo etcdctl snapshot restore /opt/backup.db --data-dir=/var/lib/etcd
sudo chown -R root:root /var/lib/etcd
sudo systemctl start kubelet
```

---

## 🎓 Key Takeaways

✅ **etcd = cluster state database** - Everything lives here  
✅ **Snapshot = point-in-time backup** - Entire cluster state  
✅ **Restore = cluster time machine** - Go back to exact state  
✅ **Always backup before upgrades** - Safety net for failures  
✅ **Test restores regularly** - Don't wait for disaster  
✅ **Store backups off-site** - Node failure protection  
✅ **Verify every snapshot** - Ensure backup validity  

## PreUmps
-> This is very much aligned with Calico and current on prem k8s wth kubeAdm
-> The structure may vary based on which kind of CNI you may use in terms of commands, but the crux will remain same.
