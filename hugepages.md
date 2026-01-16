## Understanding HugePages

Standard Linux memory pages are 4KB in size. High-performance packet processing applications require larger memory pages to:

- Reduce Translation Lookaside Buffer (TLB) misses
- Provide predictable memory allocation
- Eliminate kernel page fragmentation

HugePages allocate fixed memory blocks of 2MB or 1GB at boot time, enabling Kubernetes to schedule performance-critical workloads with guaranteed memory characteristics.

### Check Current HugePages Status

On your worker node:

```bash
grep Huge /proc/meminfo
```

A fresh system typically shows:

```
HugePages_Total: 0
```

---

## Configuring HugePages

### Determine HugePage Size

For most CNDP and DPDK workloads, **2MB HugePages** are recommended as they provide an optimal balance between performance and flexibility.

### Enable HugePages at Boot

Edit the GRUB configuration:

```bash
sudo vi /etc/default/grub
```

Modify the `GRUB_CMDLINE_LINUX` parameter:

```
GRUB_CMDLINE_LINUX="default_hugepagesz=2M hugepagesz=2M hugepages=1024"
```

This configuration allocates 1024 × 2MB = 2GB of HugePages.

Apply the changes:

```bash
sudo update-grub
sudo reboot
```

### Verify HugePage Allocation

After reboot, confirm allocation:

```bash
grep Huge /proc/meminfo
```

Expected output:

```
HugePages_Total: 1024
HugePages_Free: 1024
Hugepagesize: 2048 kB
```

Verify Kubernetes detects the HugePages:

```bash
kubectl describe node worker-1 | grep -i huge
```

You should see:

```
hugepages-2Mi: 2048Mi
```

If HugePages are not visible to Kubernetes, troubleshoot kubelet configuration before proceeding.

---

## Understanding CPU Pinning

CPU pinning ensures deterministic performance by binding container processes to specific CPU cores, preventing:

- Context switching overhead
- Cache invalidation
- Latency variability

Kubernetes enables CPU pinning only for pods with **Guaranteed Quality of Service (QoS)**, where CPU requests equal CPU limits.

### Analyze CPU Topology

Examine your node's CPU architecture:

```bash
lscpu
```

Note the total CPUs and NUMA node distribution:

```
CPU(s):              16
NUMA node0 CPU(s):   0-7
NUMA node1 CPU(s):   8-15
```

---

## Configuring CPU Manager

### Reserve System CPUs

Reserve cores for operating system and Kubernetes components to prevent resource contention.

Edit kubelet configuration:

```bash
sudo vi /var/lib/kubelet/config.yaml
```

Add or modify:

```yaml
cpuManagerPolicy: static
reservedSystemCPUs: "0-1"
```

This reserves CPUs 0-1 for system processes, leaving remaining cores available for workloads.

Restart kubelet:

```bash
sudo systemctl restart kubelet
```

### Verify CPU Manager Configuration

```bash
kubectl describe node worker-1 | grep -i cpu
```

Confirm the output includes:

```
cpuManagerPolicy: static
```

---

## System-Level Performance Tuning

### Kernel Parameters

Create a dedicated sysctl configuration:

```bash
sudo vi /etc/sysctl.d/99-cndp.conf
```

Add performance-optimized parameters:

```conf
vm.nr_hugepages=1024
kernel.sched_rt_runtime_us=-1
net.core.netdev_max_backlog=250000
net.core.rmem_max=134217728
net.core.wmem_max=134217728
```

Apply the configuration:

```bash
sudo sysctl --system
```

### CPU Frequency Scaling

Disable CPU power-saving features for consistent performance:

```bash
sudo apt install linux-tools-common -y
sudo cpupower frequency-set -g performance
```

Verify the governor setting:

```bash
cpupower frequency-info
```

### CPU Isolation (Optional)

For maximum performance, isolate specific CPUs from the kernel scheduler.

Edit GRUB configuration:

```bash
sudo vi /etc/default/grub
```

Append to `GRUB_CMDLINE_LINUX`:

```
isolcpus=2-15 nohz_full=2-15 rcu_nocbs=2-15
```

Apply changes:

```bash
sudo update-grub
sudo reboot
```

This configuration dedicates CPUs 2-15 exclusively to workload containers.

---

## Deploying a High-Performance Pod

### Pod Specification

Create a pod manifest with guaranteed resources:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cndp-pod
spec:
  nodeSelector:
    kubernetes.io/hostname: worker-1
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    resources:
      limits:
        cpu: "4"
        memory: "2Gi"
        hugepages-2Mi: "1Gi"
      requests:
        cpu: "4"
        memory: "2Gi"
        hugepages-2Mi: "1Gi"
```

Deploy the pod:

```bash
kubectl apply -f pod.yaml
```

### Resource Configuration Explained

| Parameter | Purpose |
|-----------|---------|
| `cpu: requests == limits` | Triggers Guaranteed QoS and enables CPU pinning |
| `hugepages-2Mi` | Allocates memory from the HugePage pool |
| `nodeSelector` | Ensures pod placement on the configured node |

---

## Verification Procedures

### Verify Pod QoS Class

```bash
kubectl get pod cndp-pod -o jsonpath='{.status.qosClass}'
```

Expected output: `Guaranteed`

### Verify CPU Pinning

Retrieve the pod UID:

```bash
kubectl get pod cndp-pod -o jsonpath='{.metadata.uid}'
```

On the worker node, check the assigned CPU set:

```bash
cat /sys/fs/cgroup/cpuset/kubepods.slice/*/*/cpuset.cpus
```

The output (e.g., `2-5`) indicates the specific CPUs assigned to the pod.

### Verify HugePage Usage

Execute inside the pod:

```bash
kubectl exec cndp-pod -- grep Huge /proc/meminfo
```

### Verify CPU Isolation

On the worker node:

```bash
ps -eo pid,psr,comm | grep kube
```

Confirm that system processes are not running on isolated CPUs.

---

## Advanced Networking with Multus (Optional)

For workloads requiring multiple network interfaces, SR-IOV, or dedicated data plane networking, install Multus CNI:

```bash
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml
```
