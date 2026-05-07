# Resource Requests — Practical Exercises

## Setup
```bash
kubectl create namespace reqlab
kubectl config set-context --current --namespace=reqlab
```

---

## Section A — Set Requests and See Them Reflected

### A1. Pod with explicit requests
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: small }
spec:
  containers:
  - name: c
    image: nginx
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
EOF

kubectl get pod small -o jsonpath='{.spec.containers[0].resources}' | python3 -m json.tool
```

### A2. See the QoS class
```bash
kubectl get pod small -o jsonpath='{.status.qosClass}'
# Burstable (requests but no limits)
```

---

## Section B — Pod That Won't Schedule

### B1. Request more than any node has
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: huge }
spec:
  containers:
  - name: c
    image: nginx
    resources:
      requests:
        memory: "500Gi"
EOF

kubectl get pod huge
kubectl describe pod huge | tail -8
```
Expected: `FailedScheduling: 0/N nodes are available: N Insufficient memory.`

### B2. Cleanup
```bash
kubectl delete pod huge
```

---

## Section C — Inspect Node's Allocatable

```bash
NODE=$(kubectl get pod small -o jsonpath='{.spec.nodeName}')
kubectl describe node $NODE | grep -A8 "Allocated resources"
```
You'll see: total requests / limits across all pods on this node, plus what's still free.

---

## Section D — QoS Classes

### D1. Guaranteed (requests == limits, on every container)
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: guar }
spec:
  containers:
  - name: c
    image: nginx
    resources:
      requests: { cpu: "200m", memory: "200Mi" }
      limits:   { cpu: "200m", memory: "200Mi" }
EOF

kubectl get pod guar -o jsonpath='{.status.qosClass}'
```

### D2. Burstable
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: burst }
spec:
  containers:
  - name: c
    image: nginx
    resources:
      requests: { cpu: "100m", memory: "100Mi" }
      limits:   { cpu: "200m", memory: "300Mi" }
EOF

kubectl get pod burst -o jsonpath='{.status.qosClass}'
```

### D3. BestEffort
```bash
kubectl run effort --image=nginx
kubectl get pod effort -o jsonpath='{.status.qosClass}'
```

### D4. Confirm all three
```bash
for p in guar burst effort; do
  printf "%-8s -> " $p
  kubectl get pod $p -o jsonpath='{.status.qosClass}{"\n"}'
done
```

### D5. Cleanup
```bash
kubectl delete pod guar burst effort small
```

---

## Section E — Inspect Cgroups (on the node)

For minikube:
```bash
minikube ssh
# inside the VM:
PID=$(pgrep -f nginx | head -1)
cat /proc/$PID/cgroup
ls /sys/fs/cgroup/$(awk -F: '{print $3}' /proc/$PID/cgroup)/
cat /sys/fs/cgroup/$(awk -F: '{print $3}' /proc/$PID/cgroup)/cpu.weight
exit
```
**Lesson:** the kubelet translates your CPU request into `cpu.weight` (cgroup v2) or `cpu.shares` (v1). Higher request = higher weight = more CPU during contention.

---

## Section F — LimitRange (Default Requests)

### F1. Create a LimitRange
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata: { name: defaults }
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
EOF
```

### F2. Run a pod with NO requests
```bash
kubectl run defaulted --image=nginx
kubectl get pod defaulted -o jsonpath='{.spec.containers[0].resources}' | python3 -m json.tool
```
**Expected:** the LimitRange injected the defaults. The pod is now Burstable, not BestEffort.

### F3. Cleanup
```bash
kubectl delete pod defaulted
kubectl delete limitrange defaults
```

---

## Section G — ResourceQuota

### G1. Cap the namespace
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata: { name: ns-cap }
spec:
  hard:
    requests.cpu: "1"
    requests.memory: "1Gi"
EOF
```

### G2. Try to over-budget
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: overspend }
spec:
  containers:
  - name: c
    image: nginx
    resources:
      requests: { cpu: "2", memory: "100Mi" }      # exceeds quota
EOF
```
Expected: API server rejects with `exceeded quota`.

### G3. Cleanup
```bash
kubectl delete pod overspend --ignore-not-found
kubectl delete resourcequota ns-cap
```

---

## Section H — Cleanup
```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace reqlab
```
