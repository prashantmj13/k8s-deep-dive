# Manual Pod Scheduling — Exercises

Hands-on. You will pin pods to specific nodes by setting `spec.nodeName`, hit the gotchas, and confirm the scheduler is being skipped.

## Setup

```bash
kubectl create namespace mansched
kubectl config set-context --current --namespace=mansched
kubectl get nodes
```

Single-node clusters work for most exercises; sections that need multiple nodes are marked.

When done:
```bash
kubectl delete namespace mansched
```

---

## Section A — Pin a Pod to a Specific Node

### A1. Find a node name
```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
echo $NODE
```

### A2. Create a manually-scheduled pod
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pinned
spec:
  nodeName: $NODE
  containers:
  - name: c
    image: nginx
EOF
```

### A3. Confirm the scheduler did not touch it
```bash
kubectl get pod pinned -o wide
kubectl describe pod pinned | grep -A2 Events
```
Look at the events. There should be **no** `Scheduled` event from `default-scheduler` — only `Pulled`, `Created`, `Started`. The scheduler was skipped.

---

## Section B — Pin to a Non-Existent Node

### B1. Create a pod targeting a fake node
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ghost
spec:
  nodeName: does-not-exist
  containers:
  - name: c
    image: nginx
EOF
```

### B2. Watch what happens
```bash
kubectl get pod ghost
kubectl describe pod ghost | tail -10
```
**Question:** what status does the pod have? Why?

### B3. Cleanup
```bash
kubectl delete pod ghost
```

---

## Section C — Bypass a Taint

### C1. Add a NoSchedule taint to a node
```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl taint nodes $NODE special=true:NoSchedule
```

### C2. Try to place a normal pod (should be Pending)
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: normal-pod }
spec:
  containers:
  - { name: c, image: nginx }
EOF

kubectl get pod normal-pod
# stays Pending on a single-node cluster
kubectl describe pod normal-pod | tail -5
```

### C3. Now bypass with manual scheduling
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: forced-pod }
spec:
  nodeName: $NODE
  containers:
  - { name: c, image: nginx }
EOF

kubectl get pod forced-pod -o wide
```
**Lesson:** the manual pod runs even on a tainted node. Taints are scheduler-side and were skipped.

### C4. Cleanup
```bash
kubectl delete pod forced-pod normal-pod --ignore-not-found
kubectl taint nodes $NODE special-
```

---

## Section D — Patch nodeName Onto an Existing Pending Pod

### D1. Create a pod that won't schedule (huge memory request)
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: hungry }
spec:
  containers:
  - name: c
    image: nginx
    resources:
      requests:
        memory: "500Gi"
EOF

kubectl get pod hungry
kubectl describe pod hungry | tail -5
```
The scheduler can't fit it; it's Pending forever.

### D2. Force it onto a node manually
```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl patch pod hungry -p '{"spec":{"nodeName":"'$NODE'"}}'
kubectl get pod hungry -o wide
```
**Question:** does the pod actually run? Why or why not?

### D3. Cleanup
```bash
kubectl delete pod hungry
```

---

## Section E — Confirm `nodeName` Is Immutable on Running Pods

### E1. Create and let it run
```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: stay }
spec:
  nodeName: $NODE
  containers: [{ name: c, image: nginx }]
EOF

kubectl wait --for=condition=Ready pod/stay --timeout=60s
```

### E2. Try to move it
```bash
kubectl patch pod stay -p '{"spec":{"nodeName":"some-other-node"}}'
```
**Question:** what error appears?

### E3. Cleanup
```bash
kubectl delete pod stay
```

---

## Section F — Multi-Node Cluster: Compare Scheduler vs Manual

(Requires 2+ nodes. Skip if single-node.)

### F1. Two pods, two strategies
```bash
NODES=( $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}') )
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: by-scheduler }
spec:
  containers: [{ name: c, image: nginx }]
---
apiVersion: v1
kind: Pod
metadata: { name: by-hand }
spec:
  nodeName: ${NODES[1]}
  containers: [{ name: c, image: nginx }]
EOF

kubectl get pods -o wide
```

### F2. Inspect events for both
```bash
kubectl describe pod by-scheduler | grep -A2 Events
kubectl describe pod by-hand | grep -A2 Events
```
**Compare:** the scheduler-bound pod has a `Scheduled` event; the manual one does not.

### F3. Cleanup
```bash
kubectl delete pod by-scheduler by-hand
```

---

## Section G — Watch the Scheduler Logs Confirm Skipping

### G1. Tail scheduler logs
```bash
kubectl logs -n kube-system -l component=kube-scheduler --tail=50 -f &
```
(Or the pod name directly: `kubectl logs -n kube-system kube-scheduler-<node> -f`)

### G2. Create a manual pod
```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl run manual --image=nginx --overrides='{"spec":{"nodeName":"'$NODE'"}}'
```
**Observation:** no scheduler log line for `manual` (because the scheduler doesn't see assigned pods).

### G3. Create a normal pod
```bash
kubectl run normal --image=nginx
```
**Observation:** scheduler logs show `Successfully bound pod default/normal to <node>`.

### G4. Cleanup
```bash
kubectl delete pod manual normal
kill %1 2>/dev/null
```

---

## Section H — Stretch

### H1. Override mid-flight via raw API
```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl proxy --port=8080 &
sleep 1
curl -X POST http://localhost:8080/api/v1/namespaces/mansched/pods \
  -H 'Content-Type: application/json' \
  -d '{
    "apiVersion":"v1","kind":"Pod",
    "metadata":{"name":"raw-api"},
    "spec":{"nodeName":"'$NODE'","containers":[{"name":"c","image":"nginx"}]}
  }'
kill %1
kubectl get pod raw-api -o wide
kubectl delete pod raw-api
```

### H2. Try a Deployment with a templated nodeName
You generally cannot template `nodeName` to different nodes per pod, since all replicas of a Deployment share the template. This is another reason manual scheduling is a single-pod tool, not a workload-management tool.

---

## Section I — Cleanup

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace mansched
```

Open `03-Solution.md` for expected outputs and answers.
