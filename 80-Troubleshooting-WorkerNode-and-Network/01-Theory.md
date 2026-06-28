# 80 — Troubleshooting: Worker Node Failures and Network Issues

## Part 1: Worker Node Failures

### Node Status Conditions

```bash
kubectl get nodes
kubectl describe node <node>
```

| Condition | Meaning when True |
|-----------|--------------------|
| `Ready` | kubelet is healthy and ready to accept pods |
| `MemoryPressure` | Node is running low on memory |
| `DiskPressure` | Node is running low on disk |
| `PIDPressure` | Too many processes running |
| `NetworkUnavailable` | Node's network is not correctly configured |

```bash
kubectl get node <node> -o jsonpath='{.status.conditions}' | python3 -m json.tool
```

### Node NotReady — Diagnosis Steps

```bash
# 1. Is the node reachable at all?
ping <node-ip>
ssh <node>

# 2. Is kubelet running?
ssh <node> 'sudo systemctl status kubelet'
ssh <node> 'sudo journalctl -u kubelet -f --since "10 min ago"'

# 3. Is the container runtime running?
ssh <node> 'sudo systemctl status containerd'
ssh <node> 'sudo crictl ps'

# 4. Disk space?
ssh <node> 'df -h'

# 5. Memory?
ssh <node> 'free -h'
```

### Common Node Failure Causes

| Symptom | Cause | Fix |
|---------|-------|-----|
| `NotReady`, kubelet down | kubelet crashed/stopped | `systemctl restart kubelet`, check logs |
| `NotReady`, DiskPressure | Disk full (often from old images/logs) | `crictl rmi --prune`, clean logs, expand disk |
| `NotReady`, certificate error in kubelet logs | kubelet client cert expired | Renew certs, restart kubelet |
| Node disappeared entirely | VM terminated, network partition | Check cloud console / hypervisor |
| Pods evicted from node | MemoryPressure/DiskPressure eviction | Investigate resource usage, add capacity |

### Node Eviction Behavior

When a node goes `NotReady`, the control plane waits `pod-eviction-timeout` (default 5 min) before evicting and rescheduling its pods elsewhere:

```bash
kubectl get pods -o wide | grep <bad-node>
# After ~5 min if node stays down:
kubectl get pods -o wide   # pods now show as rescheduled to other nodes
```

### Draining and Recovering a Node

```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --force

# Fix the underlying issue (disk, kubelet config, hardware), then:
kubectl uncordon <node>
```

### Recovering kubelet on a Broken Node

```bash
# Check kubelet config
cat /var/lib/kubelet/config.yaml

# Check kubelet's client certificate
ls -la /var/lib/kubelet/pki/

# Restart fully
sudo systemctl daemon-reload
sudo systemctl restart kubelet
sudo systemctl status kubelet
```

---

## Part 2: Network Troubleshooting

### Network Troubleshooting Decision Tree

```
Can't reach a Service?
   ↓
Does the Service have Endpoints?  → kubectl get endpoints <svc>
   ↓ NO                              ↓ YES
Check pod labels match               Is kube-proxy healthy on the node?
selector / readiness probe              ↓
                                    Can you reach the pod IP directly?
                                         ↓ NO
                                    CNI / network policy problem
```

### Step 1 — Does DNS resolve?

```bash
kubectl run dnsutils --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3 --rm -it -- sh
nslookup kubernetes.default
nslookup my-svc.my-namespace.svc.cluster.local
cat /etc/resolv.conf
```

If DNS fails:
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl get svc -n kube-system kube-dns
```

### Step 2 — Does the Service have Endpoints?

```bash
kubectl get endpoints my-svc
kubectl get endpointslices -l kubernetes.io/service-name=my-svc
```

If empty: check the Service's `selector` actually matches pod labels, and check pod readiness:
```bash
kubectl get pods --show-labels
kubectl get svc my-svc -o jsonpath='{.spec.selector}'
```

### Step 3 — Can you reach the pod IP directly (bypass Service)?

```bash
kubectl get pod <pod> -o jsonpath='{.status.podIP}'
kubectl run debug --image=busybox --rm -it -- wget -qO- --timeout=3 http://<pod-ip>:<port>
```

If pod IP works but Service doesn't → kube-proxy issue.
If pod IP also fails → CNI/network policy issue.

### Step 4 — Check kube-proxy

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
kubectl logs -n kube-system <kube-proxy-pod>

# On the node:
sudo iptables -t nat -L KUBE-SERVICES -n | grep <service-ip>
```

### Step 5 — Check CNI / Pod Networking

```bash
# Is the CNI plugin's DaemonSet healthy?
kubectl get pods -n kube-system | grep -E "calico|flannel|weave|cilium"

# Check pod's network namespace directly
kubectl exec <pod> -- ip addr
kubectl exec <pod> -- ip route
kubectl exec <pod> -- cat /etc/resolv.conf

# Cross-node connectivity test
kubectl run net-test-a --image=busybox --overrides='{"spec":{"nodeName":"node1"}}' -- sleep 3600
kubectl run net-test-b --image=busybox --overrides='{"spec":{"nodeName":"node2"}}' -- sleep 3600
kubectl exec net-test-a -- ping -c3 $(kubectl get pod net-test-b -o jsonpath='{.status.podIP}')
```

### Step 6 — Check NetworkPolicy Blocking Traffic

```bash
kubectl get networkpolicies -A
kubectl describe networkpolicy -n <namespace>
# Temporarily remove suspected policy to confirm it's the cause:
kubectl delete networkpolicy <policy> -n <namespace> --dry-run=client
```

### Common Network Failure Causes Summary

| Symptom | Likely Cause |
|---------|-------------|
| DNS lookup fails entirely | CoreDNS pods down, or DNS Service has no endpoints |
| Service unreachable, pod IP works | kube-proxy not running/misconfigured on the node |
| Pod-to-pod fails even with no policies, same node | CNI plugin issue on that node |
| Pod-to-pod fails cross-node only | CNI overlay/routing issue (VXLAN/BGP) between nodes |
| Works sometimes, not always | DNS caching, intermittent CNI agent restarts, or flapping readiness |
| Specific pods can't reach specific others | NetworkPolicy misconfiguration |

### Useful One-Liners

```bash
# Quick connectivity test pod
kubectl run tmp-shell --rm -i --tty --image=busybox -- sh

# Check if a port is open from inside the cluster
kubectl run tmp-shell --rm -i --tty --image=busybox -- nc -zv <host> <port>

# Full DNS + network debug image
kubectl run netshoot --rm -i --tty --image=nicolaka/netshoot -- bash
```
