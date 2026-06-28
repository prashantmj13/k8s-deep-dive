# 79 — Troubleshooting: Application Failures and Control Plane Failures

## Part 1: Application Failures

### Troubleshooting Workflow

```
kubectl get pods                    → what's the STATUS?
       ↓
kubectl describe pod <name>         → Events section — WHY?
       ↓
kubectl logs <pod> [-c container]   → application-level error?
       ↓
kubectl exec -it <pod> -- sh        → inspect from inside
```

### Common Pod States and Causes

| Status | Likely Cause |
|--------|-------------|
| `Pending` | Unschedulable (no resources, taints, no matching node) or image pulling |
| `ImagePullBackOff` | Wrong image name/tag, missing imagePullSecret, registry auth failure |
| `CrashLoopBackOff` | App crashes immediately — bad config, missing dependency, wrong command |
| `Error` / `OOMKilled` | Container exceeded memory limit, or exited with non-zero code |
| `ContainerCreating` (stuck) | Volume mount issue, CNI/IP allocation failure |
| `Running` but `0/1` Ready | Failing readiness probe |
| `Completed` then restarting | restartPolicy mismatch with workload type (e.g. Job vs bare pod) |

### Diagnosing Pending Pods

```bash
kubectl describe pod <pod> | grep -A10 Events
# Look for: "Insufficient cpu", "node(s) didn't match Pod's node affinity",
#           "node(s) had taint ... that the pod didn't tolerate"

kubectl get nodes -o wide
kubectl describe node <node> | grep -A5 "Allocated resources"
```

### Diagnosing CrashLoopBackOff

```bash
kubectl logs <pod> --previous     # logs from the PREVIOUS crashed instance
kubectl describe pod <pod> | grep -A5 "Last State"
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState}'
```

Common root causes:
- Wrong `command`/`args` overriding the image's intended entrypoint
- Missing required environment variable or mounted Secret/ConfigMap
- App's health check endpoint not implemented, causing liveness restarts
- Dependency (DB, queue) unreachable at startup with no retry logic

### Diagnosing ImagePullBackOff

```bash
kubectl describe pod <pod> | grep -A3 "Failed"
# "rpc error: ... unauthorized" → missing/wrong imagePullSecret
# "manifest unknown" → wrong tag
# "no such host" → wrong registry URL or DNS issue
```

### Diagnosing OOMKilled

```bash
kubectl describe pod <pod> | grep -A3 "Last State"
# Reason: OOMKilled, Exit Code: 137
```
Fix: increase `resources.limits.memory`, or fix a memory leak in the app.

### Diagnosing Readiness/Liveness Failures

```bash
kubectl describe pod <pod> | grep -A5 "Liveness\|Readiness"
kubectl get events --field-selector involvedObject.name=<pod>
```

### Application-level Debugging

```bash
kubectl exec -it <pod> -- sh
# inside: check env vars, test connectivity, check mounted files
env | sort
cat /etc/config/app.properties
wget -qO- http://dependent-service:8080/health
nslookup dependent-service
```

---

## Part 2: Control Plane Failures

### Key Control Plane Components

```
kube-apiserver  ← everything goes through here
kube-controller-manager
kube-scheduler
etcd
```

### Checking Control Plane Health

```bash
# Static pods (kubeadm clusters) — check they're running
kubectl get pods -n kube-system

# If kubectl itself is unreachable, check directly on the control plane node:
sudo crictl ps | grep -E "kube-apiserver|etcd|controller-manager|scheduler"
sudo systemctl status kubelet

# Check static pod manifests are correctly placed
ls /etc/kubernetes/manifests/
```

### API Server Down

Symptoms: `kubectl` commands hang or return `connection refused` / `the server could not find the requested resource`.

```bash
# On the control plane node, check kubelet logs (it manages static pods)
journalctl -u kubelet -f

# Check the API server static pod manifest for typos/bad flags
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Check if the container is even attempting to start
sudo crictl ps -a | grep apiserver
sudo crictl logs <container-id>
```

Common causes:
- Bad flag added to the static pod manifest (typo causes the container to crash-loop)
- Certificate expired (`x509: certificate has expired`)
- etcd unreachable (API server depends on etcd — check etcd first)
- Disk full on the control plane node

### etcd Down

```bash
sudo crictl ps -a | grep etcd
sudo crictl logs <etcd-container-id>

# Check etcd health directly
ETCDCTL_API=3 etcdctl endpoint health \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

Common causes:
- Disk full (`etcdserver: mvcc: database space exceeded` — NOSPACE alarm)
- Clock skew between control plane nodes in HA setups (Raft is timing-sensitive)
- Certificate expiry

```bash
# Check for NOSPACE alarm
etcdctl alarm list
# Fix: free space, then:
etcdctl alarm disarm
etcdctl defrag
```

### Scheduler/Controller-Manager Down

Symptoms: new pods stay `Pending` forever (scheduler down) or Deployments don't reconcile replicas (controller-manager down), but existing pods keep running fine (API server + kubelet still functional).

```bash
kubectl get pods -n kube-system | grep -E "scheduler|controller-manager"
kubectl logs -n kube-system kube-scheduler-controlplane
kubectl logs -n kube-system kube-controller-manager-controlplane
```

### Certificate Expiry (very common cause of "sudden" control plane failure)

```bash
kubeadm certs check-expiration
# If expired:
kubeadm certs renew all
# Then restart static pods by moving manifests out and back, or restart kubelet
sudo systemctl restart kubelet
```

### General Control Plane Recovery Checklist

1. Is the node itself reachable (ping, ssh)?
2. Is kubelet running? (`systemctl status kubelet`)
3. Are static pod manifests present and valid YAML?
4. Are certificates valid (not expired)?
5. Is etcd healthy and has disk space?
6. Are there any recent manual edits to static pod manifests or `/etc/kubernetes/`?
7. Check `journalctl -u kubelet` for the specific error driving the failure
