
## What is a DaemonSet?

A **DaemonSet** ensures that a copy of a Pod runs on all (or selected) nodes in the cluster. When nodes are added, Pods are automatically added to them. When nodes are removed, those Pods are garbage collected.

**Key Characteristics:**
- Runs **one pod per node** (not replica count-based)
- Automatically deploys to new nodes
- Survives pod deletions (recreates on same node)
- Managed by Controller Manager, not scheduler replica logic

**Common Use Cases:**
- Network plugins (Calico, Flannel)
- Log collectors (Fluentd, Filebeat)
- Monitoring agents (Prometheus Node Exporter)
- Storage daemons (Ceph, GlusterFS)

---

## DaemonSet Commands Explained

### 1. List DaemonSets

```bash
kubectl get daemonsets -n kube-system
```

**What it does:**
- Lists all DaemonSets in the `kube-system` namespace
- Shows NAME, DESIRED (number of nodes), CURRENT (running pods), READY, UP-TO-DATE, AVAILABLE

**Example Output:**
```
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
calico-node   3         3         3       3            3
kube-proxy    3         3         3       3            3
```

**Why `-n kube-system`?**
- System-level DaemonSets (networking, proxy) run in `kube-system`
- User DaemonSets would be in `default` or custom namespaces

---

### 2. View DaemonSet Pods Distribution

```bash
kubectl get pods -n kube-system -o wide | grep kube-proxy
```

**What it does:**
- `-o wide`: Shows additional columns including NODE and IP
- `grep kube-proxy`: Filters only kube-proxy pods
- Proves one pod exists per node

**Example Output:**
```
NAME               READY   STATUS    NODE           IP
kube-proxy-abc12   1/1     Running   control-plane  192.168.1.10
kube-proxy-def34   1/1     Running   worker-1       192.168.1.11
kube-proxy-ghi56   1/1     Running   worker-2       192.168.1.12
```

**Key Insight:** Each node has exactly one kube-proxy pod.

---

### 3. Create a DaemonSet

```bash
kubectl apply -f ds-demo.yaml
```

**YAML Breakdown:**

```yaml
apiVersion: apps/v1          # API version for DaemonSet
kind: DaemonSet              # Resource type
metadata:
  name: ds-demo              # DaemonSet name
spec:
  selector:                  # How DaemonSet finds its pods
    matchLabels:
      app: ds-demo
  template:                  # Pod template (identical to Deployment)
    metadata:
      labels:
        app: ds-demo         # Must match selector
    spec:
      containers:
      - name: logger
        image: busybox       # Lightweight container
        command: ["sh", "-c", "while true; do echo running-on-$(hostname); sleep 10; done"]
```

**Key Fields:**
- `selector.matchLabels`: Links DaemonSet to Pods
- `template`: Pod specification (same as Deployment/StatefulSet)
- `command`: Prints hostname every 10 seconds (proof of node assignment)

---

### 4. Verify DaemonSet Deployment

```bash
kubectl get pods -o wide
```

**What to observe:**
- Number of pods = Number of nodes
- Each pod on different node
- All pods have same label (`app=ds-demo`)

---

### 5. Test DaemonSet Self-Healing

```bash
kubectl delete pod <ds-pod-on-worker-1>
kubectl get pods
```

**What happens:**
1. Pod terminates
2. DaemonSet controller detects missing pod on worker-1
3. **New pod created on same node** (not just any node)

**Difference from Deployment:**
- Deployment: Recreates pod on any available node
- DaemonSet: Recreates pod on **original node only**

---

## DaemonSet vs Deployment

| Feature | DaemonSet | Deployment |
|---------|-----------|------------|
| **Replica Strategy** | One per node | Fixed replica count |
| **Scaling** | Automatic with nodes | Manual/HPA |
| **Pod Placement** | All nodes (or selected) | Scheduler decides |
| **Use Case** | Node-level services | Application workloads |
| **Pod Deletion** | Recreates on same node | Recreates on any node |

---

# Job vs CronJob

## Understanding Jobs

A **Job** creates one or more Pods and ensures that a specified number of them successfully terminate. Jobs track successful completions.

**Key Characteristics:**
- Runs to completion (doesn't restart forever)
- Retries on failure (configurable)
- Pod remains after completion (for logs)
- Exit code 0 = success

**Use Cases:**
- Database migrations
- Batch processing
- One-time scripts
- Data transformations

---

## Understanding CronJobs

A **CronJob** creates Jobs on a repeating schedule (cron format).

**Key Characteristics:**
- Creates Jobs at scheduled times
- Each execution = new Job = new Pod(s)
- History limit configurable
- Based on Unix cron syntax

**Use Cases:**
- Backups
- Report generation
- Cleanup tasks
- Scheduled scraping

---

## Job Commands Explained

### 1. Create a Simple Job

```bash
kubectl create job hello-job --image=busybox -- echo "HELLO FROM JOB"
```

**Command Breakdown:**
- `create job`: Imperative Job creation
- `hello-job`: Job name
- `--image=busybox`: Container image
- `--`: Separator between kubectl args and container command
- `echo "HELLO FROM JOB"`: Command to run in container

**What happens:**
1. Job created
2. Pod spawned
3. Container runs `echo` command
4. Pod status changes to **Completed**
5. Job marked as successful

---

### 2. Check Job Status

```bash
kubectl get jobs
```

**Example Output:**
```
NAME        COMPLETIONS   DURATION   AGE
hello-job   1/1           5s         30s
```

**Fields Explained:**
- `COMPLETIONS`: successful/desired (1/1 = done)
- `DURATION`: Time to complete
- `AGE`: Time since creation

---

### 3. View Job Pods

```bash
kubectl get pods
```

**Example Output:**
```
NAME              READY   STATUS      RESTARTS   AGE
hello-job-abc12   0/1     Completed   0          1m
```

**Status Values:**
- `Running`: Job in progress
- `Completed`: Job succeeded
- `Error`: Job failed (will retry)
- `CrashLoopBackOff`: Repeated failures

---

### 4. View Job Logs

```bash
kubectl logs <job-pod-name>
```

**Why logs persist:**
- Completed pods remain until manually deleted
- Allows post-mortem debugging
- Use `--tail=50` for last 50 lines
- Use `-f` for follow mode (if still running)

---

### 5. Create a Failing Job (Retry Demonstration)

```bash
kubectl create job fail-job --image=busybox -- sh -c "exit 1"
```

**What it does:**
- Container exits with code 1 (failure)
- Job controller detects failure
- Creates new pod (retry)
- Repeats until success or backoff limit

**Check retry behavior:**
```bash
kubectl get pods
```

You'll see multiple pods:
```
fail-job-abc12   0/1     Error       0          1m
fail-job-def34   0/1     Error       0          45s
fail-job-ghi56   0/1     Running     0          10s
```

**Default Retry Limit:** 6 attempts (configurable via `spec.backoffLimit`)

---

## CronJob Commands Explained

### 1. Create a CronJob

```bash
kubectl apply -f cron.yaml
```

**YAML Breakdown:**

```yaml
apiVersion: batch/v1         # CronJob API version
kind: CronJob                # Resource type
metadata:
  name: time-printer
spec:
  schedule: "*/1 * * * *"    # Cron expression (every minute)
  jobTemplate:               # Job specification (nested)
    spec:
      template:              # Pod specification (double nested)
        spec:
          containers:
          - name: printer
            image: busybox
            command: ["sh", "-c", "date"]
          restartPolicy: OnFailure  # Required for Jobs
```

**Cron Schedule Format:**
```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6)
│ │ │ │ │
* * * * *
```

**Common Patterns:**
- `*/1 * * * *` - Every minute
- `0 * * * *` - Every hour
- `0 0 * * *` - Daily at midnight
- `0 0 * * 0` - Weekly on Sunday
- `0 0 1 * *` - Monthly on 1st

---

### 2. List CronJobs

```bash
kubectl get cronjobs
```

**Example Output:**
```
NAME           SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
time-printer   */1 * * * *   False     1        15s             5m
```

**Fields Explained:**
- `SCHEDULE`: Cron expression
- `SUSPEND`: Paused or not
- `ACTIVE`: Currently running Jobs
- `LAST SCHEDULE`: When last Job was created

---

### 3. View CronJob-Created Jobs

```bash
kubectl get jobs
```

**Example Output:**
```
NAME                  COMPLETIONS   DURATION   AGE
time-printer-28934712 1/1           3s         1m
time-printer-28934713 1/1           2s         30s
```

**Naming Convention:** `<cronjob-name>-<unix-timestamp>`

---

### 4. View CronJob Logs

```bash
kubectl logs <job-pod>
```

**Example Output:**
```
Mon Jan  7 10:35:00 UTC 2026
```

**Tip:** Use labels to get latest:
```bash
kubectl logs -l job-name=time-printer-28934713
```

---

## Job vs CronJob: Critical Differences

| Aspect | Job | CronJob |
|--------|-----|---------|
| **Execution** | Runs once immediately | Runs on schedule |
| **Creation** | Manual trigger | Automatic (time-based) |
| **YAML Structure** | `kind: Job` | `kind: CronJob` (wraps Job) |
| **Use Case** | One-time tasks | Recurring tasks |
| **Pod Lifecycle** | Completes and stops | New pod each run |
| **History** | Single execution | Multiple executions |
| **Schedule Field** | ❌ Not present | ✅ Required |
| **Retry Behavior** | Retries until success | Each run is independent |

---

## Job/CronJob Best Practices

### Job Configuration

```yaml
spec:
  completions: 3            # Run 3 successful completions
  parallelism: 2            # Run 2 pods in parallel
  backoffLimit: 4           # Retry 4 times on failure
  activeDeadlineSeconds: 600 # Timeout after 10 minutes
```

### CronJob Configuration

```yaml
spec:
  successfulJobsHistoryLimit: 3  # Keep 3 successful Jobs
  failedJobsHistoryLimit: 1      # Keep 1 failed Job
  concurrencyPolicy: Forbid      # Don't allow overlapping
```

**Concurrency Policies:**
- `Allow`: Multiple Jobs can run concurrently
- `Forbid`: Skip new Job if previous still running
- `Replace`: Cancel old Job, start new one

---

# iptables and kube-proxy

## What is kube-proxy?

**kube-proxy** is a network proxy that runs on each node. It maintains network rules that allow communication to Pods from inside or outside the cluster.

**Key Facts:**
- Does NOT proxy traffic (misleading name!)
- Programs Linux kernel networking (iptables/ipvs)
- Translates Service IPs to Pod IPs
- Runs as DaemonSet

---

## How Service Networking Works

```
Client Request
    ↓
Service ClusterIP (10.96.x.x) [VIRTUAL - doesn't exist on any interface]
    ↓
iptables rules (KUBE-SERVICES → KUBE-SVC-xxx → KUBE-SEP-xxx)
    ↓
Pod IP (192.168.x.x) [REAL - assigned to pod interface]
```

---

## iptables Commands Explained

### 1. Create Service for Testing

```bash
kubectl create deployment web --image=nginx --replicas=2
```

**What it does:**
- Creates Deployment named `web`
- Uses nginx image
- Runs 2 replica pods

```bash
kubectl expose deployment web --port=80
```

**What it does:**
- Creates Service named `web`
- Maps to Deployment `web`
- Exposes port 80
- Assigns ClusterIP (virtual)

```bash
kubectl get svc
```

**Example Output:**
```
NAME   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
web    ClusterIP   10.96.45.78   <none>        80/TCP    1m
```

**Note the ClusterIP:** This IP doesn't exist on any network interface!

---

### 2. Inspect iptables Rules

**SSH into worker node first:**
```bash
ssh user@worker-node-ip
```

**View KUBE chains:**
```bash
sudo iptables -t nat -L -n | grep KUBE
```

**Command Breakdown:**
- `iptables`: Linux firewall utility
- `-t nat`: NAT (Network Address Translation) table
- `-L`: List rules
- `-n`: Numeric output (don't resolve hostnames)
- `grep KUBE`: Filter Kubernetes-related chains

**Output Example:**
```
Chain KUBE-SERVICES (2 references)
Chain KUBE-POSTROUTING (1 references)
Chain KUBE-MARK-MASQ (20 references)
Chain KUBE-SVC-XXXXXX (1 references)
Chain KUBE-SEP-YYYYYY (1 references)
```

**Chain Purposes:**
- `KUBE-SERVICES`: Entry point for all service traffic
- `KUBE-SVC-*`: Service-specific rules (load balancing)
- `KUBE-SEP-*`: Service Endpoint (actual Pod IPs)
- `KUBE-MARK-MASQ`: SNAT rules for external traffic

---

### 3. Trace Service to Pod Flow

**Step 1: View Service Entry**
```bash
sudo iptables -t nat -L KUBE-SERVICES -n
```

**Example Output:**
```
target              prot  destination
KUBE-SVC-ABC123     tcp   10.96.45.78:80
```

**Translation:** Traffic to `10.96.45.78:80` → jump to `KUBE-SVC-ABC123`

---

**Step 2: View Load Balancing Rules**
```bash
sudo iptables -t nat -L KUBE-SVC-ABC123 -n
```

**Example Output:**
```
target              probability
KUBE-SEP-XYZ111     0.50000000000  (50%)
KUBE-SEP-XYZ222     0.50000000000  (50%)
```

**Translation:** 50% chance to each backend pod (round-robin load balancing)

---

**Step 3: View Actual Pod IP**
```bash
sudo iptables -t nat -L KUBE-SEP-XYZ111 -n
```

**Example Output:**
```
target     prot  destination
DNAT       tcp   to:192.168.1.50:80
```

**Translation:** Rewrite destination to actual Pod IP `192.168.1.50:80`

---

### Complete Flow Diagram

```
Request to 10.96.45.78:80
         ↓
   KUBE-SERVICES (match service IP)
         ↓
   KUBE-SVC-ABC123 (load balance)
         ↓ (50% chance)
   KUBE-SEP-XYZ111 (pod 1)
         ↓
   DNAT to 192.168.1.50:80
```

---

### 4. Break Service Networking (Proof)

```bash
kubectl delete pod -n kube-system -l k8s-app=kube-proxy
```

**What it does:**
- Deletes all kube-proxy pods
- Label selector: `-l k8s-app=kube-proxy`
- Temporarily removes iptables management

**Test from any pod:**
```bash
kubectl run test --image=busybox -it --rm -- sh
wget http://10.96.45.78  # Service IP - FAILS ❌
ping 192.168.1.50        # Pod IP - WORKS ✅
exit
```

**Why:**
- Service IP → requires iptables rules → kube-proxy manages them
- Pod IP → direct CNI routing → doesn't need kube-proxy

**Recovery:**
- kube-proxy runs as DaemonSet
- Pods automatically recreate
- iptables rules restored

---

## iptables Modes

kube-proxy supports three modes:

### 1. iptables (default)
- Uses Linux iptables rules
- Kernel-space packet processing
- Low overhead
- No real load balancing (random selection)

### 2. ipvs
- Uses Linux IPVS (IP Virtual Server)
- Better performance at scale
- True load balancing algorithms (rr, lc, sh)
- Requires kernel modules

### 3. userspace (legacy)
- Proxies in userspace
- Slow (packet copying)
- Deprecated

---

## Key Networking Concepts

### ClusterIP vs Pod IP

| Property | ClusterIP | Pod IP |
|----------|-----------|--------|
| **Type** | Virtual (iptables) | Real (interface) |
| **Stability** | Stable (doesn't change) | Ephemeral (changes on recreate) |
| **Scope** | Cluster-wide | Single pod |
| **Managed By** | kube-proxy | CNI plugin |
| **Routing** | iptables/ipvs | CNI overlay/routing |

### Service Types

- **ClusterIP**: Internal only (default)
- **NodePort**: External via node IP:port
- **LoadBalancer**: Cloud provider load balancer
- **ExternalName**: DNS CNAME redirect

---

## Troubleshooting Commands

### Check kube-proxy logs
```bash
kubectl logs -n kube-system -l k8s-app=kube-proxy
```

### View iptables save/restore
```bash
sudo iptables-save | grep KUBE
```

### Check kube-proxy mode
```bash
kubectl get cm -n kube-system kube-proxy -o yaml | grep mode
```

### Test service connectivity
```bash
kubectl run test --image=nicolaka/netshoot -it --rm -- bash
curl http://service-name.namespace.svc.cluster.local
```

---

## Summary: Command Quick Reference

### DaemonSet
```bash
kubectl get daemonsets -n kube-system
kubectl get pods -o wide | grep <daemonset-name>
kubectl apply -f daemonset.yaml
kubectl delete pod <daemonset-pod>
```

### Job
```bash
kubectl create job <name> --image=<image> -- <command>
kubectl get jobs
kubectl get pods --show-all
kubectl logs <job-pod>
kubectl delete job <name>
```

### CronJob
```bash
kubectl apply -f cronjob.yaml
kubectl get cronjobs
kubectl get jobs
kubectl create job --from=cronjob/<name> <manual-job-name>
```

### iptables
```bash
sudo iptables -t nat -L -n | grep KUBE
sudo iptables -t nat -L KUBE-SERVICES -n
sudo iptables -t nat -L KUBE-SVC-<hash> -n
sudo iptables -t nat -L KUBE-SEP-<hash> -n
```

---

## Final Concepts

### Control Flow Hierarchy

```
CronJob creates → Job creates → Pod (one-time execution)
DaemonSet creates → Pod (per-node persistent)
Deployment creates → ReplicaSet creates → Pod (count-based persistent)
```

### When to Use What

- **DaemonSet**: Node-level agents (logging, monitoring, networking)
- **Job**: One-time tasks (migrations, batch processing)
- **CronJob**: Scheduled tasks (backups, cleanups)
- **Deployment**: Long-running applications
- **StatefulSet**: Stateful applications (databases)

---



