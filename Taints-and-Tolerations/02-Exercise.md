# Taints and Tolerations — Practical Exercises

## Setup
```bash
kubectl create namespace tainlab
kubectl config set-context --current --namespace=tainlab
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
echo "Working with node: $NODE"
```

When done:
```bash
kubectl delete namespace tainlab
kubectl taint nodes $NODE gpu- 2>/dev/null
kubectl taint nodes $NODE special- 2>/dev/null
kubectl taint nodes $NODE quarantine- 2>/dev/null
```

---

## Section A — Inspect Existing Taints

### A1. See what's already there
```bash
kubectl describe node $NODE | grep -A2 Taints
kubectl get node $NODE -o jsonpath='{.spec.taints}' | python3 -m json.tool 2>/dev/null
```
On most clusters, you'll see no taints (workers) or `node-role.kubernetes.io/control-plane:NoSchedule` (control plane in single-node minikube).

---

## Section B — Add a NoSchedule Taint

### B1. Taint the node
```bash
kubectl taint nodes $NODE gpu=true:NoSchedule
kubectl describe node $NODE | grep -A2 Taints
```

### B2. Try to schedule a normal pod (single-node cluster)
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: regular }
spec:
  containers: [{ name: c, image: nginx }]
EOF

kubectl get pod regular
kubectl describe pod regular | tail -6
```
**Expected (single-node):** Pod is `Pending`. Events show `0/1 nodes are available: 1 node(s) had untolerated taint {gpu: true}`.

### B3. Now create a tolerating pod
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: tolerating }
spec:
  tolerations:
  - key: gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  containers: [{ name: c, image: nginx }]
EOF

kubectl get pods
```
**Expected:** `tolerating` runs; `regular` still Pending.

### B4. Cleanup this section
```bash
kubectl delete pod regular tolerating
kubectl taint nodes $NODE gpu-
```

---

## Section C — Existing Pods Are Unaffected by NoSchedule

### C1. Schedule a pod first
```bash
kubectl run before --image=nginx
kubectl wait --for=condition=Ready pod/before --timeout=60s
```

### C2. Now taint the node
```bash
kubectl taint nodes $NODE special=true:NoSchedule
kubectl get pod before -o wide
```
**Expected:** the pod keeps running. NoSchedule only blocks NEW scheduling.

### C3. Cleanup
```bash
kubectl delete pod before
kubectl taint nodes $NODE special-
```

---

## Section D — NoExecute Evicts Existing Pods

### D1. Schedule pods, some with toleration and some without
```bash
kubectl run plain --image=nginx
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: tough }
spec:
  tolerations:
  - key: emergency
    operator: Exists
    effect: NoExecute
  containers: [{ name: c, image: nginx }]
EOF

kubectl wait --for=condition=Ready pod/plain --timeout=60s
kubectl wait --for=condition=Ready pod/tough --timeout=60s
kubectl get pods
```

### D2. Apply a NoExecute taint
```bash
kubectl taint nodes $NODE emergency=true:NoExecute
kubectl get pods -w
# Press Ctrl+C after seeing the eviction
```
**Expected:** `plain` is evicted (Terminating then gone). `tough` keeps running.

### D3. Cleanup
```bash
kubectl delete pod tough --ignore-not-found
kubectl taint nodes $NODE emergency-
```

---

## Section E — tolerationSeconds for Graceful Eviction

### E1. Schedule a pod with finite toleration
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: timed }
spec:
  tolerations:
  - key: maint
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 30
  containers: [{ name: c, image: nginx }]
EOF

kubectl wait --for=condition=Ready pod/timed --timeout=60s
```

### E2. Apply the taint and time it
```bash
kubectl taint nodes $NODE maint=true:NoExecute
date
kubectl get pod timed -w
# Press Ctrl+C ~45 seconds later
date
```
**Expected:** the pod runs for ~30 seconds after the taint, then is evicted.

### E3. Cleanup
```bash
kubectl taint nodes $NODE maint-
```

---

## Section F — `Exists` Operator and the "Tolerate Everything" Pattern

### F1. The catch-all toleration
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: catchall }
spec:
  tolerations:
  - operator: Exists           # tolerate EVERY taint
  containers: [{ name: c, image: nginx }]
EOF

kubectl taint nodes $NODE one=1:NoSchedule
kubectl taint nodes $NODE two=2:NoSchedule
kubectl get pod catchall -o wide
```
**Expected:** runs anyway because it tolerates everything.

### F2. Cleanup
```bash
kubectl delete pod catchall
kubectl taint nodes $NODE one-
kubectl taint nodes $NODE two-
```

---

## Section G — Combine with nodeSelector for Real "Dedicated Node"

The pattern: taint to KEEP OTHERS OUT, label + nodeSelector to PULL YOUR PODS IN.

### G1. Taint and label the node
```bash
kubectl taint nodes $NODE gpu=true:NoSchedule
kubectl label nodes $NODE workload=gpu
```

### G2. Create a pod that both tolerates and selects
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: gpu-only }
spec:
  nodeSelector:
    workload: gpu
  tolerations:
  - { key: gpu, operator: Equal, value: "true", effect: NoSchedule }
  containers: [{ name: c, image: nginx }]
EOF

kubectl get pod gpu-only -o wide
```

### G3. Compare with toleration-only (no nodeSelector) on a multi-node cluster
On a cluster with multiple nodes, a toleration alone permits the GPU node but doesn't *prefer* it. The pod might still land elsewhere if other nodes have free capacity. nodeSelector is what actually pulls it.

### G4. Cleanup
```bash
kubectl delete pod gpu-only
kubectl taint nodes $NODE gpu-
kubectl label nodes $NODE workload-
```

---

## Section H — DaemonSets and Control-Plane Taints

DaemonSet pods need to land on **every** node, including control-plane nodes that have a `NoSchedule` taint. Inspect a system DaemonSet:

```bash
kubectl get ds -n kube-system kube-proxy -o yaml | grep -A20 tolerations
```
**Expected:** broad `Exists`-style tolerations covering control-plane and all conditions.

---

## Section I — The `kubectl drain` Command

`drain` is taint-and-evict in one step.

### I1. Cordon (just adds the taint)
```bash
kubectl cordon $NODE
kubectl describe node $NODE | grep -A2 Taints
# node.kubernetes.io/unschedulable:NoSchedule appears
kubectl get node $NODE
# STATUS: SchedulingDisabled
```

### I2. Drain (cordon + evict)
```bash
kubectl run shorty --image=nginx
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --force
kubectl get pods
```
**Expected:** the pod is evicted because drain adds a NoSchedule taint and politely deletes existing pods.

### I3. Reverse
```bash
kubectl uncordon $NODE
```

---

## Section J — Cleanup
```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace tainlab
# Make sure you didn't leave taints behind
kubectl describe node $NODE | grep -A5 Taints
```

Open `03-Solution.md` for expected outputs.
