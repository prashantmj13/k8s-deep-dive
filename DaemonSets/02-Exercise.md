# DaemonSets — Practical Exercises

## Setup
```bash
kubectl create namespace dslab
kubectl config set-context --current --namespace=dslab
```

---

## Section A — Inspect Existing DaemonSets

```bash
kubectl get ds -A
kubectl describe ds -n kube-system kube-proxy | head -40
kubectl get ds -n kube-system kube-proxy -o yaml | grep -A20 tolerations
```
Note the broad tolerations — kube-proxy must run on every node.

---

## Section B — Create a Simple DaemonSet

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata: { name: my-agent }
spec:
  selector:
    matchLabels: { app: my-agent }
  template:
    metadata:
      labels: { app: my-agent }
    spec:
      tolerations:
      - operator: Exists
      containers:
      - name: c
        image: nginx
        resources:
          requests: { cpu: 50m, memory: 64Mi }
EOF

kubectl get ds my-agent
kubectl get pods -l app=my-agent -o wide
```
Pod count = number of nodes (single-node cluster -> 1 pod).

---

## Section C — Restrict to Specific Nodes

### C1. Label a node
```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl label nodes $NODE role=worker
```

### C2. Patch the DS to target only those
```bash
kubectl patch ds my-agent --type=merge -p '{"spec":{"template":{"spec":{"nodeSelector":{"role":"worker"}}}}}'
kubectl get pods -l app=my-agent -o wide
```

### C3. Remove the label, see the pod disappear
```bash
kubectl label nodes $NODE role-
kubectl get pods -l app=my-agent -o wide
# Pod is terminated because no node matches anymore
```

---

## Section D — Rolling Update

### D1. Set rolling update params and update image
```bash
kubectl patch ds my-agent --type=merge -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"maxUnavailable":1}},"template":{"spec":{"nodeSelector":null}}}}'
kubectl set image ds/my-agent c=nginx:1.27
kubectl rollout status ds/my-agent
kubectl rollout history ds/my-agent
```

### D2. Restart all DS pods without changing image
```bash
kubectl rollout restart ds/my-agent
kubectl rollout status ds/my-agent
```

---

## Section E — OnDelete Update Strategy

```bash
kubectl patch ds my-agent --type=merge -p '{"spec":{"updateStrategy":{"type":"OnDelete"}}}'
kubectl set image ds/my-agent c=nginx:1.26
kubectl get pods -l app=my-agent -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.spec.containers[0].image}{"\n"}{end}'
# image is still 1.27 — controller did not auto-replace
kubectl delete pods -l app=my-agent
kubectl get pods -l app=my-agent -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}'
# now 1.26
```

---

## Section F — Cleanup
```bash
kubectl delete ds my-agent
kubectl config set-context --current --namespace=default
kubectl delete namespace dslab
```
