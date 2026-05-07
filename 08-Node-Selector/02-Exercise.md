# nodeSelector — Practical Exercises

## Setup
```bash
kubectl create namespace nslab
kubectl config set-context --current --namespace=nslab
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
echo "Node: $NODE"
```

When done:
```bash
kubectl label nodes $NODE disktype- environment- workload- 2>/dev/null
kubectl delete namespace nslab
```

---

## Section A — Inspect Built-in Labels

```bash
kubectl get nodes --show-labels
kubectl get node $NODE -o jsonpath='{.metadata.labels}' | python3 -m json.tool
kubectl get nodes -L kubernetes.io/os,kubernetes.io/arch,topology.kubernetes.io/zone
```

---

## Section B — Add Custom Labels

```bash
kubectl label nodes $NODE disktype=ssd
kubectl label nodes $NODE environment=prod
kubectl get nodes --show-labels | grep $NODE
```

---

## Section C — Schedule with nodeSelector

### C1. Create a pod that requires our labels
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: ssd-only }
spec:
  nodeSelector:
    disktype: ssd
    environment: prod
  containers: [{ name: c, image: nginx }]
EOF

kubectl get pod ssd-only -o wide
kubectl describe pod ssd-only | grep -A2 Node-Selectors
```

### C2. Now demand a label that doesn't exist
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: nonexistent }
spec:
  nodeSelector:
    disktype: nvme            # we set ssd, not nvme
  containers: [{ name: c, image: nginx }]
EOF

kubectl get pod nonexistent
kubectl describe pod nonexistent | tail -8
```
**Expected:** Pending. Events show `0/N nodes are available: N node(s) didn't match Pod's node affinity/selector.`

### C3. Cleanup
```bash
kubectl delete pod ssd-only nonexistent --ignore-not-found
```

---

## Section D — Update Labels Mid-Flight

### D1. Create a pod that fits
```bash
kubectl run rolling --image=nginx --overrides='{"spec":{"nodeSelector":{"disktype":"ssd"}}}'
kubectl get pod rolling -o wide
kubectl wait --for=condition=Ready pod/rolling --timeout=60s
```

### D2. Remove the label from the node
```bash
kubectl label nodes $NODE disktype-
kubectl get pod rolling -o wide
```
**Question:** is the pod still running? Why?

### D3. Cleanup
```bash
kubectl delete pod rolling
kubectl label nodes $NODE disktype=ssd
```

---

## Section E — Multi-Label OR (Doesn't Work With nodeSelector)

### E1. Try the impossible
nodeSelector is AND-only. There is no built-in OR. The work-around is to label different nodes with the same key=value:

```bash
# (Multi-node clusters only)
kubectl label nodes $NODE flavor=fastdisk
# kubectl label nodes worker-2 flavor=fastdisk    # also tag a hypothetical node-2

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: fast }
spec:
  nodeSelector:
    flavor: fastdisk
  containers: [{ name: c, image: nginx }]
EOF

kubectl get pod fast -o wide
```

For real OR (`flavor in (fastdisk, fastnet)`), you need **nodeAffinity** — see that folder.

### E2. Cleanup
```bash
kubectl delete pod fast
kubectl label nodes $NODE flavor-
```

---

## Section F — Combine With Taint/Toleration ("Dedicated Nodes")

### F1. Set up the node
```bash
kubectl taint nodes $NODE workload=ml:NoSchedule
kubectl label nodes $NODE workload=ml
```

### F2. The right way to target
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: ml-job }
spec:
  nodeSelector:
    workload: ml
  tolerations:
  - { key: workload, operator: Equal, value: ml, effect: NoSchedule }
  containers: [{ name: c, image: python:3.12-slim, command: ["sleep","3600"] }]
EOF

kubectl get pod ml-job -o wide
```

### F3. Confirm a non-tolerating pod is rejected
```bash
kubectl run regular --image=nginx
kubectl get pod regular
kubectl describe pod regular | tail -5
```
On a single-node cluster, `regular` stays Pending — it doesn't tolerate the taint.

### F4. Cleanup
```bash
kubectl delete pod ml-job regular --ignore-not-found
kubectl taint nodes $NODE workload-
kubectl label nodes $NODE workload-
```

---

## Section G — Use Built-in Labels for Portability

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: portable }
spec:
  nodeSelector:
    kubernetes.io/os: linux
    kubernetes.io/arch: amd64
  containers: [{ name: c, image: nginx }]
EOF

kubectl get pod portable -o wide
```
**Lesson:** these labels are guaranteed to be present, so manifests using them are portable across clouds and on-prem clusters.

---

## Section H — Inspect From the Pod Side

```bash
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.spec.nodeSelector}{"\n"}{end}'
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.nodeSelector}{"\n"}{end}' | grep -v "^[^ ]* $"
```

---

## Section I — Cleanup
```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace nslab
kubectl label nodes $NODE disktype- environment- workload- 2>/dev/null
```

Open `03-Solution.md` for expected outputs.
