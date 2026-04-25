# ReplicaSet — Practical Exercises

Hands-on. You will create ReplicaSets, scale them, kill pods, adopt and orphan pods, watch the controller react, and intentionally hit the gotchas.

## Prerequisites

```bash
kubectl get nodes      # cluster up?
kubectl version        # client + server
```

A clean namespace makes life easier:
```bash
kubectl create namespace rslab
kubectl config set-context --current --namespace=rslab
```

When you finish the exercises:
```bash
kubectl delete namespace rslab
```

---

## Section A — Create Your First ReplicaSet

### A1. Write the manifest
```bash
cat > rs-web.yaml <<'EOF'
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:1.25
        ports:
        - containerPort: 80
EOF
```

### A2. Apply and inspect
```bash
kubectl apply -f rs-web.yaml
kubectl get rs
kubectl get pods -l app=web -o wide
```

### A3. Look at the RS object
```bash
kubectl describe rs web
kubectl get rs web -o yaml | head -40
```
Find: `replicas` (desired), `fullyLabeledReplicas`, `readyReplicas`, `availableReplicas`.

### A4. Confirm pod ownership
```bash
POD=$(kubectl get pods -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl get pod $POD -o jsonpath='{.metadata.ownerReferences}' | python3 -m json.tool 2>/dev/null || \
kubectl get pod $POD -o jsonpath='{.metadata.ownerReferences}'
```
You should see `kind: ReplicaSet`, `name: web`, `controller: true`, `blockOwnerDeletion: true`.

---

## Section B — Watch Self-Healing Live

### B1. Delete a pod and watch
Open two terminals.

Terminal 1:
```bash
kubectl get pods -l app=web -w
```

Terminal 2:
```bash
POD=$(kubectl get pods -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $POD
```

In terminal 1 you should see the pod move to `Terminating`, and within a few seconds a new pod appear and reach `Running`. Press Ctrl+C to stop watching.

### B2. How fast did it react? Look at the events
```bash
kubectl get events --sort-by='.lastTimestamp' | tail -15
```
Find lines like `SuccessfulCreate ... Created pod: web-xxxxx` — those come from the ReplicaSet controller.

### B3. Delete ALL the pods at once
```bash
kubectl delete pods -l app=web
kubectl get pods -l app=web -w
# All three are recreated in parallel
```

---

## Section C — Scaling

### C1. Scale up via kubectl
```bash
kubectl scale rs web --replicas=5
kubectl get rs web
kubectl get pods -l app=web
```

### C2. Scale up by editing the manifest
```bash
sed -i 's/replicas: 3/replicas: 7/' rs-web.yaml
# (on macOS use: sed -i '' 's/replicas: 5/replicas: 7/' rs-web.yaml)
kubectl apply -f rs-web.yaml
kubectl get rs web
```

### C3. Scale down — observe which pods get deleted
```bash
kubectl scale rs web --replicas=2
kubectl get pods -l app=web -w
# Stop watching when count is 2
```
**Question:** which pods got picked for deletion? Hint: `kubectl get pods -l app=web --sort-by=.metadata.creationTimestamp`

### C4. Reset
```bash
kubectl scale rs web --replicas=3
```

---

## Section D — Selectors in Detail

### D1. List pods by label
```bash
kubectl get pods -l app=web
kubectl get pods -l 'app in (web)'
kubectl get pods -l 'app!=web'
kubectl get pods --show-labels
```

### D2. Add a label to a pod (still matches)
```bash
POD=$(kubectl get pods -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl label pod $POD env=prod
kubectl get pod $POD --show-labels
kubectl get rs web        # still 3, no churn
```
**Why no churn?** The selector `app=web` still matches. Extra labels are fine.

### D3. The set-based selector form
```bash
cat > rs-set.yaml <<'EOF'
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchExpressions:
    - { key: app,  operator: In, values: [api, api-v2] }
    - { key: tier, operator: NotIn, values: [debug] }
  template:
    metadata:
      labels:
        app: api
        tier: backend
    spec:
      containers:
      - { name: api, image: nginx:1.25 }
EOF
kubectl apply -f rs-set.yaml
kubectl get rs api
kubectl get pods -l app=api
```

### D4. Cleanup the set-based one
```bash
kubectl delete rs api
```

---

## Section E — Orphaning a Pod (remove its label)

### E1. Remove the matching label from one pod
```bash
POD=$(kubectl get pods -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl get rs web      # 3 ready
kubectl label pod $POD app-       # the trailing dash REMOVES the label
kubectl get pods --show-labels
kubectl get rs web
```

### E2. What just happened?
- The orphaned pod still exists but no longer matches the selector.
- The RS sees 2 matching pods, wants 3, creates a new one.
- You now have 4 pods total: 3 owned by the RS, 1 ownerless.

### E3. Use this trick for debugging
"Take this misbehaving pod out of rotation without losing it." Now you can `kubectl exec` into the orphan, inspect, then delete it manually when you're done.

### E4. Cleanup the orphan
```bash
kubectl delete pod $POD
kubectl get pods -l app=web
```

---

## Section F — Adoption

### F1. Create a labelless pod, then add the label later
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: stranger
spec:
  containers:
  - { name: c, image: nginx:1.25 }
EOF

kubectl get pod stranger --show-labels
kubectl get rs web
```

### F2. Add the matching label
```bash
kubectl label pod stranger app=web
kubectl get pods -l app=web
kubectl get rs web
```

### F3. Watch what happens
The RS now sees 4 pods matching the selector. It deletes one to bring the count back to 3.

```bash
kubectl get pods -l app=web -w
```

The pod chosen for deletion may or may not be `stranger` — depends on creation timestamp and readiness. Look at `ownerReferences` on the survivors:
```bash
for p in $(kubectl get pods -l app=web -o jsonpath='{.items[*].metadata.name}'); do
  echo "=== $p ==="
  kubectl get pod $p -o jsonpath='{.metadata.ownerReferences[*].name}'
  echo
done
```

---

## Section G — The Big Gotcha: Editing the Template Image

### G1. Change the image in the manifest
```bash
sed -i 's/nginx:1.25/nginx:1.27/' rs-web.yaml
kubectl apply -f rs-web.yaml
kubectl get rs web
kubectl get rs web -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### G2. Look at the running pods
```bash
kubectl get pods -l app=web -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.spec.containers[0].image}{"\n"}{end}'
```
**Question:** what image are the existing pods running?

### G3. Force a rotation by deleting the pods
```bash
kubectl delete pods -l app=web
kubectl get pods -l app=web -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.spec.containers[0].image}{"\n"}{end}'
```

**Lesson:** ReplicaSets do NOT roll updates. Deployments do.

---

## Section H — Cascade vs Orphan Deletion

### H1. The default — cascade
```bash
kubectl delete rs web
kubectl get pods -l app=web
```
Pods are gone with the RS.

### H2. Recreate, then delete with `--cascade=orphan`
```bash
kubectl apply -f rs-web.yaml
kubectl get pods -l app=web

kubectl delete rs web --cascade=orphan
kubectl get rs
kubectl get pods -l app=web
```
The RS is gone but the pods are still running, now without an owner.

### H3. Check ownerReferences on the orphans
```bash
POD=$(kubectl get pods -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl get pod $POD -o jsonpath='{.metadata.ownerReferences}'
# empty / null
```

### H4. A new RS will adopt them
```bash
kubectl apply -f rs-web.yaml
kubectl get rs web
kubectl get pods -l app=web
kubectl get pod $POD -o jsonpath='{.metadata.ownerReferences[*].name}'
# now points to the new RS
```

### H5. Cleanup
```bash
kubectl delete rs web
```

---

## Section I — Two RSs with Overlapping Selectors (anti-pattern)

### I1. Create two RSs that both want pods with `app=conflict`
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: ReplicaSet
metadata: { name: rs-a }
spec:
  replicas: 2
  selector: { matchLabels: { app: conflict } }
  template:
    metadata: { labels: { app: conflict, owner: a } }
    spec:
      containers: [{ name: c, image: nginx:1.25 }]
---
apiVersion: apps/v1
kind: ReplicaSet
metadata: { name: rs-b }
spec:
  replicas: 2
  selector: { matchLabels: { app: conflict } }
  template:
    metadata: { labels: { app: conflict, owner: b } }
    spec:
      containers: [{ name: c, image: nginx:1.27 }]
EOF
```

### I2. Watch the chaos
```bash
kubectl get rs
kubectl get pods -l app=conflict -w
# You'll see pods being created and deleted by both RSs
```
Each RS thinks it has too many pods (because it also counts the other RS's pods), so they keep deleting each other's work.

### I3. Cleanup
```bash
kubectl delete rs rs-a rs-b
```

**Lesson:** never have two controllers with overlapping selectors. This is also why you should never set the labels on a Deployment-managed pod manually.

---

## Section J — Performance: Bulk Scale

### J1. Scale to 30 and time it
```bash
kubectl apply -f rs-web.yaml
time kubectl scale rs web --replicas=30
kubectl get pods -l app=web --no-headers | wc -l
```

### J2. Scale to 0 and back
```bash
kubectl scale rs web --replicas=0
kubectl get rs web
kubectl get pods -l app=web      # all gone

kubectl scale rs web --replicas=5
kubectl get pods -l app=web
```
**Useful pattern:** scaling a workload to 0 effectively "pauses" it without losing the spec.

---

## Section K — End-to-End: Watch the API Server

In one terminal, watch raw events:
```bash
kubectl get events --watch
```

In another terminal, run through these:
```bash
kubectl apply -f rs-web.yaml
kubectl scale rs web --replicas=4
kubectl delete pod $(kubectl get pods -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl scale rs web --replicas=2
kubectl delete rs web
```

Each action produces visible API events. You will see lines from the `replicaset-controller` source: `SuccessfulCreate`, `SuccessfulDelete`, etc.

---

## Section L — Stretch / Advanced

### L1. Use `kubectl --v=8` to see the API calls
```bash
kubectl --v=8 scale rs web --replicas=4 2>&1 | grep -E '(GET|PATCH|POST)' | head
```

### L2. Find which controller owns a given pod via raw API
```bash
kubectl get --raw /api/v1/namespaces/rslab/pods | python3 -m json.tool | head -50
```

### L3. Drain a node and watch RS react
On a multi-node cluster:
```bash
NODE=$(kubectl get pods -l app=web -o jsonpath='{.items[0].spec.nodeName}')
kubectl cordon $NODE
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data
kubectl get pods -l app=web -o wide
kubectl uncordon $NODE
```

### L4. Try to change the selector — see the API reject it
```bash
kubectl edit rs web
# change selector.matchLabels.app from web to webX, save
# you should get an error: "field is immutable"
```

### L5. Inspect controller logs on a kubeadm cluster
```bash
kubectl logs -n kube-system kube-controller-manager-<node> | grep -i replicaset | tail -20
```

---

## Final Cleanup

```bash
kubectl delete rs --all
kubectl delete pods --all
kubectl config set-context --current --namespace=default
kubectl delete namespace rslab
```

Open `03-Solution.md` for expected outputs and the answers to every question.
