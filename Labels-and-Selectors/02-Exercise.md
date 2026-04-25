# Labels and Selectors — Practical Exercises

## Setup
```bash
kubectl create namespace lblab
kubectl config set-context --current --namespace=lblab
```

When done:
```bash
kubectl delete namespace lblab
```

---

## Section A — Create the Test Population

### A1. Eight pods with varied labels
```bash
for spec in \
  "p1 app=web env=prod tier=frontend" \
  "p2 app=web env=prod tier=frontend" \
  "p3 app=web env=staging tier=frontend" \
  "p4 app=web env=prod tier=backend" \
  "p5 app=api env=prod tier=backend" \
  "p6 app=api env=staging tier=backend" \
  "p7 app=db env=prod tier=data" \
  "p8 app=db env=staging tier=data"; do
  set -- $spec
  NAME=$1; shift
  kubectl run $NAME --image=nginx --labels="$(echo $@ | tr ' ' ',')"
done
```

### A2. See them all
```bash
kubectl get pods --show-labels
kubectl get pods -L app,env,tier
```

---

## Section B — Equality-Based Selectors

### B1. Single match
```bash
kubectl get pods -l app=web
# Expected: p1, p2, p3, p4
```

### B2. AND of two
```bash
kubectl get pods -l app=web,env=prod
# Expected: p1, p2, p4
```

### B3. Not equal
```bash
kubectl get pods -l env!=prod
# Expected: p3, p6, p8
```

### B4. Combine
```bash
kubectl get pods -l 'app=web,env=prod,tier=frontend'
# Expected: p1, p2
```

---

## Section C — Set-Based Selectors

### C1. In
```bash
kubectl get pods -l 'app in (web,api)'
# Expected: p1..p6
```

### C2. NotIn
```bash
kubectl get pods -l 'tier notin (data)'
# Expected: p1..p6
```

### C3. Exists
```bash
kubectl label pod p1 owner=team-a
kubectl get pods -l 'owner'         # any pod that has the key, regardless of value
```

### C4. DoesNotExist
```bash
kubectl get pods -l '!owner'        # everything that does not have owner
```

### C5. Combine equality + set
```bash
kubectl get pods -l 'app in (web,api), env=prod'
# Expected: p1, p2, p4, p5
```

---

## Section D — Bulk Operations With Selectors

### D1. Add a label to every web pod
```bash
kubectl label pods -l app=web team=alpha
kubectl get pods -L team
```

### D2. Delete every staging pod
```bash
kubectl delete pods -l env=staging
kubectl get pods
```

### D3. Re-create the staging ones for further exercises
```bash
kubectl run p3 --image=nginx --labels="app=web,env=staging,tier=frontend"
kubectl run p6 --image=nginx --labels="app=api,env=staging,tier=backend"
kubectl run p8 --image=nginx --labels="app=db,env=staging,tier=data"
```

---

## Section E — Selectors in a Service

### E1. Create a Service that selects only `app=web`
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata: { name: web-svc }
spec:
  selector:
    app: web
  ports:
  - { port: 80, targetPort: 80 }
EOF

kubectl get endpoints web-svc
```
Should list 4 IPs (p1, p2, p3, p4 — the pods with `app=web`).

### E2. Tighten the selector
```bash
kubectl patch svc web-svc --type=merge -p '{"spec":{"selector":{"app":"web","env":"prod"}}}'
kubectl get endpoints web-svc
```
Now only 3 endpoints (p1, p2, p4).

### E3. What if the selector matches nothing?
```bash
kubectl patch svc web-svc --type=merge -p '{"spec":{"selector":{"app":"nope"}}}'
kubectl get endpoints web-svc
# ENDPOINTS column shows <none>
```
**Lesson:** `kubectl get endpoints <svc>` is the fastest way to debug "my Service isn't routing traffic."

### E4. Cleanup
```bash
kubectl delete svc web-svc
```

---

## Section F — Add / Update / Remove Labels

```bash
# Add (errors if exists without --overwrite)
kubectl label pod p1 stage=v1

# Update
kubectl label pod p1 stage=v2 --overwrite

# Remove (trailing dash)
kubectl label pod p1 stage-

# All-at-once filter + label
kubectl label pods -l env=prod region=us-east
kubectl get pods -L region
```

---

## Section G — Annotations Are Not Selectable

```bash
kubectl annotate pod p1 description="this is the primary frontend"
kubectl get pod p1 -o jsonpath='{.metadata.annotations}'

# Try to select on annotation:
kubectl get pods -l description=primary
# returns nothing — annotations are NOT indexed
```

---

## Section H — Use the Canonical Labels

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: canonical
  labels:
    app.kubernetes.io/name: nginx
    app.kubernetes.io/instance: nginx-demo
    app.kubernetes.io/version: "1.27"
    app.kubernetes.io/component: web
    app.kubernetes.io/part-of: demo-stack
    app.kubernetes.io/managed-by: kubectl
spec:
  containers: [{ name: c, image: nginx:1.27 }]
EOF

kubectl get pods -l app.kubernetes.io/name=nginx
```

---

## Section I — Cleanup
```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace lblab
```
