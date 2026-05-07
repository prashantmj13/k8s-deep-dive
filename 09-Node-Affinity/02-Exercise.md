# Node Affinity — Practical Exercises

## Setup
```bash
kubectl create namespace nalab
kubectl config set-context --current --namespace=nalab
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl label nodes $NODE disktype=ssd zone=us-east-1a tier=high
```

When done:
```bash
kubectl label nodes $NODE disktype- zone- tier- gpu- 2>/dev/null
kubectl delete namespace nalab
```

---

## Section A — Required (hard) Affinity

### A1. Pod requires zone us-east-1a OR us-east-1b
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: zoned }
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - { key: zone, operator: In, values: [us-east-1a, us-east-1b] }
  containers: [{ name: c, image: nginx }]
EOF

kubectl get pod zoned -o wide
```

### A2. Force a Pending pod with a key that doesn't exist
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: ghost-zone }
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - { key: zone, operator: In, values: [us-west-2c] }
  containers: [{ name: c, image: nginx }]
EOF

kubectl describe pod ghost-zone | tail -8
```

### A3. Cleanup
```bash
kubectl delete pod zoned ghost-zone
```

---

## Section B — Operators

### B1. Exists
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: needs-disk }
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - { key: disktype, operator: Exists }
  containers: [{ name: c, image: nginx }]
EOF
kubectl get pod needs-disk -o wide
```

### B2. NotIn
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: not-dev }
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - { key: tier, operator: NotIn, values: [dev, test] }
  containers: [{ name: c, image: nginx }]
EOF
kubectl get pod not-dev -o wide
```

### B3. DoesNotExist
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: not-deprecated }
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - { key: deprecated, operator: DoesNotExist }
  containers: [{ name: c, image: nginx }]
EOF
kubectl get pod not-deprecated -o wide
```

### B4. Cleanup
```bash
kubectl delete pod needs-disk not-dev not-deprecated
```

---

## Section C — Preferred (soft) Affinity

### C1. Prefer SSD nodes (will land on any node if no SSD nodes exist)
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: prefer-ssd }
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - { key: disktype, operator: In, values: [ssd] }
  containers: [{ name: c, image: nginx }]
EOF
kubectl get pod prefer-ssd -o wide
```

### C2. With a non-existent label — pod still schedules (soft fallback)
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: prefer-fake }
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - { key: gpu, operator: Exists }
  containers: [{ name: c, image: nginx }]
EOF
kubectl get pod prefer-fake -o wide
```
**Lesson:** soft preference never blocks scheduling. Compare with the required version, which would have stayed Pending.

### C3. Cleanup
```bash
kubectl delete pod prefer-ssd prefer-fake
```

---

## Section D — Required + Preferred Together

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: combo }
spec:
  affinity:
    nodeAffinity:
      # MUST be on a Linux node
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - { key: kubernetes.io/os, operator: In, values: [linux] }
      # PREFER SSD over HDD
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - { key: disktype, operator: In, values: [ssd] }
      # PREFER zone us-east-1a a little less
      - weight: 20
        preference:
          matchExpressions:
          - { key: zone, operator: In, values: [us-east-1a] }
  containers: [{ name: c, image: nginx }]
EOF

kubectl get pod combo -o wide
```

---

## Section E — Multiple `nodeSelectorTerms` = OR

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: or-terms }
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        # Term 1: ssd AND zone-a
        - matchExpressions:
          - { key: disktype, operator: In, values: [ssd] }
          - { key: zone, operator: In, values: [us-east-1a] }
        # Term 2: zone-b alone
        - matchExpressions:
          - { key: zone, operator: In, values: [us-east-1b] }
  containers: [{ name: c, image: nginx }]
EOF
```
A node passes if (ssd AND us-east-1a) OR (us-east-1b). Term-level OR; expression-level AND.

```bash
kubectl delete pod combo or-terms
```

---

## Section F — Cleanup
```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace nalab
kubectl label nodes $NODE disktype- zone- tier- 2>/dev/null
```
