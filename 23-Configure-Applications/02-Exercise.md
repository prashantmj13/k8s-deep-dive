# Configuring Applications — Exercises

This folder is the umbrella overview. The deep-dive exercises live in the per-mechanism folders:

- `Commands-and-Arguments/02-Exercise.md`
- `Environment-Variables/02-Exercise.md`
- `Secrets/02-Exercise.md`
- `ConfigMaps/02-Exercise.md` (created as the "Additional Resources" folder in this set)

But here's a single integration exercise that uses all of them.

## Setup
```bash
kubectl create namespace cfglab
kubectl config set-context --current --namespace=cfglab
```

---

## Section A — Build a Pod That Uses All Five Mechanisms

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata: { name: app-config }
data:
  greeting: "Hello, world"
  app.conf: |
    log_level: info
    listen: 8080
---
apiVersion: v1
kind: Secret
metadata: { name: app-secret }
type: Opaque
stringData:
  api-key: super-secret-key-12345
---
apiVersion: v1
kind: Pod
metadata: { name: full-config }
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c"]
    args: ["env | sort; echo ----; cat /etc/app/app.conf; echo ----; cat /etc/secret/api-key; sleep 3600"]
    env:
    - name: LOG_LEVEL
      value: debug
    - name: API_KEY
      valueFrom:
        secretKeyRef: { name: app-secret, key: api-key }
    envFrom:
    - configMapRef: { name: app-config }
    volumeMounts:
    - { name: config-file, mountPath: /etc/app, readOnly: true }
    - { name: secret-file, mountPath: /etc/secret, readOnly: true }
    - { name: cache, mountPath: /var/cache }
  volumes:
  - name: config-file
    configMap: { name: app-config }
  - name: secret-file
    secret: { secretName: app-secret }
  - name: cache
    emptyDir: {}
EOF

kubectl wait --for=condition=Ready pod/full-config --timeout=60s
kubectl logs full-config | head -40
```

You should see:
- All env vars: `LOG_LEVEL=debug`, `API_KEY=super-secret-key-12345`, `greeting=Hello, world`, `app.conf=...` (the whole file is the value), plus the standard ones K8s injects.
- The ConfigMap file `/etc/app/app.conf` printed.
- The Secret file `/etc/secret/api-key` printed.

---

## Section B — Modify a Mounted ConfigMap and Watch It Update

```bash
kubectl edit configmap app-config
# change greeting: to "Hello, updated world"
sleep 70                                # mounted files take up to ~1 min
kubectl exec full-config -- cat /etc/app/app.conf | head
# the mounted file reflects the change
kubectl exec full-config -- printenv greeting
# but the env var still shows the OLD value (env vars don't auto-update)
```

---

## Section C — Cleanup
```bash
kubectl delete pod full-config
kubectl delete configmap app-config
kubectl delete secret app-secret
kubectl config set-context --current --namespace=default
kubectl delete namespace cfglab
```
