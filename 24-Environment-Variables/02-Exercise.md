# Environment Variables — Exercises

## Setup
```bash
kubectl create namespace envlab
kubectl config set-context --current --namespace=envlab
```

---

## Section A — Literal Values

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: literal }
spec:
  containers:
  - name: c
    image: busybox
    command: ["sh","-c","env | sort; sleep 3600"]
    env:
    - { name: LOG_LEVEL, value: info }
    - { name: PORT, value: "8080" }
    - { name: GREETING, value: "Hello World" }
EOF

kubectl wait --for=condition=Ready pod/literal --timeout=60s
kubectl logs literal | grep -E '^(LOG_LEVEL|PORT|GREETING)='
```

---

## Section B — ConfigMap Source

```bash
kubectl create configmap demo --from-literal=database=postgres --from-literal=cache=redis

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: from-cm }
spec:
  containers:
  - name: c
    image: busybox
    command: ["sh","-c","env | sort; sleep 3600"]
    env:
    - name: DB
      valueFrom: { configMapKeyRef: { name: demo, key: database } }
    envFrom:
    - { configMapRef: { name: demo }, prefix: APP_ }
EOF

kubectl wait --for=condition=Ready pod/from-cm --timeout=60s
kubectl logs from-cm | grep -E '^(DB|APP_)='
```
You'll see `DB=postgres` and bulk-imported `APP_database=postgres`, `APP_cache=redis`.

Cleanup:
```bash
kubectl delete pod from-cm
kubectl delete configmap demo
```

---

## Section C — Secret Source

```bash
kubectl create secret generic db-creds \
  --from-literal=user=admin \
  --from-literal=password=s3cret

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: from-secret }
spec:
  containers:
  - name: c
    image: busybox
    command: ["sh","-c","env | grep -E '^(USER|PASS)='; sleep 3600"]
    env:
    - name: USER
      valueFrom: { secretKeyRef: { name: db-creds, key: user } }
    - name: PASS
      valueFrom: { secretKeyRef: { name: db-creds, key: password } }
EOF

kubectl wait --for=condition=Ready pod/from-secret --timeout=60s
kubectl logs from-secret
```

Cleanup:
```bash
kubectl delete pod from-secret
kubectl delete secret db-creds
```

---

## Section D — Downward API

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: downward }
spec:
  containers:
  - name: c
    image: busybox
    command: ["sh","-c","env | grep -E '^(POD|NODE)_'; sleep 3600"]
    env:
    - { name: POD_NAME, valueFrom: { fieldRef: { fieldPath: metadata.name } } }
    - { name: POD_NAMESPACE, valueFrom: { fieldRef: { fieldPath: metadata.namespace } } }
    - { name: POD_IP, valueFrom: { fieldRef: { fieldPath: status.podIP } } }
    - { name: NODE_NAME, valueFrom: { fieldRef: { fieldPath: spec.nodeName } } }
EOF

kubectl wait --for=condition=Ready pod/downward --timeout=60s
kubectl logs downward
```

Cleanup:
```bash
kubectl delete pod downward
```

---

## Section E — Resource Field

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: from-resources }
spec:
  containers:
  - name: c
    image: busybox
    command: ["sh","-c","env | grep -E '^(CPU|MEM)_'; sleep 3600"]
    resources:
      requests: { cpu: "200m", memory: "256Mi" }
      limits:   { cpu: "500m", memory: "512Mi" }
    env:
    - { name: CPU_REQUEST, valueFrom: { resourceFieldRef: { resource: requests.cpu, divisor: "1m" } } }
    - { name: MEM_LIMIT, valueFrom: { resourceFieldRef: { resource: limits.memory, divisor: "1Mi" } } }
EOF

kubectl wait --for=condition=Ready pod/from-resources --timeout=60s
kubectl logs from-resources
```
Expected: `CPU_REQUEST=200`, `MEM_LIMIT=512`.

Cleanup:
```bash
kubectl delete pod from-resources
```

---

## Section F — Updates Don't Propagate

```bash
kubectl create configmap mutable --from-literal=greeting=hello

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: stale }
spec:
  containers:
  - name: c
    image: busybox
    command: ["sh","-c","while true; do echo greeting=$GREETING; sleep 5; done"]
    env:
    - { name: GREETING, valueFrom: { configMapKeyRef: { name: mutable, key: greeting } } }
EOF

kubectl logs -f stale &
sleep 10

# Edit the ConfigMap
kubectl edit configmap mutable
# change greeting: hello to greeting: bonjour, save

# Wait, then watch the logs — the env var DOES NOT update
sleep 30
kill %1

# Force restart to pick up the new value
kubectl delete pod stale
```
**Lesson:** `valueFrom: configMapKeyRef` is a one-time snapshot at pod start. Mounted volumes update; env vars don't.

Cleanup:
```bash
kubectl delete configmap mutable
```

---

## Section G — Cleanup
```bash
kubectl delete pod literal --ignore-not-found
kubectl config set-context --current --namespace=default
kubectl delete namespace envlab
```
