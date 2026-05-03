# Priority Classes — Practical Exercises

## Setup
```bash
kubectl create namespace prilab
kubectl config set-context --current --namespace=prilab
```

---

## Section A — Inspect Built-ins
```bash
kubectl get priorityclasses
kubectl get priorityclass system-node-critical -o yaml
```

---

## Section B — Create Three Custom Tiers

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: { name: low-prio }
value: -100
description: "Batch / preemptible"
preemptionPolicy: Never
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: { name: mid-prio }
value: 100
description: "Default"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: { name: high-prio }
value: 1000
description: "Latency-sensitive"
EOF

kubectl get pc
```

---

## Section C — Force Preemption

### C1. Fill the cluster with low-priority pods
Adjust memory request to roughly half of node allocatable:
```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
ALLOC=$(kubectl get node $NODE -o jsonpath='{.status.allocatable.memory}')
echo "Node alloc memory: $ALLOC"

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: { name: low-fill }
spec:
  replicas: 4
  selector: { matchLabels: { app: low } }
  template:
    metadata: { labels: { app: low } }
    spec:
      priorityClassName: low-prio
      containers:
      - name: c
        image: nginx
        resources:
          requests: { memory: "1Gi", cpu: "200m" }
EOF

kubectl get pods -l app=low -o wide
```
On a small cluster, some pods land and others may stay Pending — that's fine.

### C2. Submit a high-priority pod that won't fit otherwise
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: vip }
spec:
  priorityClassName: high-prio
  containers:
  - name: c
    image: nginx
    resources:
      requests: { memory: "1Gi", cpu: "200m" }
EOF

kubectl get pods -w
# Watch events: low-fill pods start being evicted
```

### C3. See the preemption events
```bash
kubectl get events --sort-by='.lastTimestamp' | tail -10
kubectl describe pod vip | tail -10
```
**Expected:** events showing `Preempted ... by prilab/vip on node ... preemption: ` referencing each evicted low-priority pod.

### C4. Cleanup
```bash
kubectl delete deploy low-fill
kubectl delete pod vip --ignore-not-found
```

---

## Section D — preemptionPolicy: Never

### D1. Create a pod that won't preempt
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: polite }
spec:
  priorityClassName: low-prio        # has policy: Never
  containers:
  - name: c
    image: nginx
    resources:
      requests: { memory: "100Gi" }    # absurd, won't fit
EOF

kubectl describe pod polite | tail -5
```
**Expected:** `FailedScheduling`, but no preemption happens — the pod just waits. With `PreemptLowerPriority` it would have tried (and still failed in this case).

### D2. Cleanup
```bash
kubectl delete pod polite
```

---

## Section E — Global Default

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: { name: default-pri }
value: 50
globalDefault: true
description: "Default for pods without priorityClassName"
EOF

kubectl run test --image=nginx
kubectl get pod test -o jsonpath='{.spec.priorityClassName}{"\n"}{.spec.priority}'
```
**Expected:** the pod inherited `default-pri` and priority 50 even though we didn't ask.

Cleanup:
```bash
kubectl delete pod test
kubectl delete pc default-pri low-prio mid-prio high-prio
```

---

## Section F — Cleanup
```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace prilab
```
