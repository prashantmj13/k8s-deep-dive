# 25 — Secrets | Solutions & Expected Outputs

---

## Exercise 1 — Namespace

```bash
kubectl create namespace secrets-lab
# Output: namespace/secrets-lab created

kubectl config set-context --current --namespace=secrets-lab
# Output: Context "..." modified.
```

---

## Exercise 2 — Create a Secret imperatively

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=supersecret123
# Output: secret/db-secret created

kubectl get secret db-secret
```

**Expected output:**
```
NAME        TYPE     DATA   AGE
db-secret   Opaque   2      5s
```

```bash
kubectl describe secret db-secret
```

**Expected output:**
```
Name:         db-secret
Namespace:    secrets-lab
Type:         Opaque

Data
====
DB_PASSWORD:  16 bytes
DB_USER:      5 bytes
```

> **Answer:** `describe` shows byte counts, not values. Kubernetes intentionally hides Secret data from describe output to prevent accidental exposure in logs/terminals. The data is still base64-encoded in etcd (not encrypted unless Encryption at Rest is enabled).

---

## Exercise 3 — Decode a Secret

```bash
kubectl get secret db-secret -o jsonpath='{.data.DB_USER}' | base64 --decode
# Output: admin

kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode
# Output: supersecret123
```

> **Key insight:** base64 is **encoding**, not **encryption**. Anyone with `kubectl get secret` RBAC permission can decode values. Use RBAC carefully and enable Encryption at Rest for production clusters.

---

## Exercise 4 — Secret from a file

```bash
echo -n "my-super-private-key-content" > /tmp/api.key

kubectl create secret generic api-key-secret \
  --from-file=api.key=/tmp/api.key
# Output: secret/api-key-secret created

kubectl get secret api-key-secret -o yaml
```

**Expected output (partial):**
```yaml
apiVersion: v1
data:
  api.key: bXktc3VwZXItcHJpdmF0ZS1rZXktY29udGVudA==
kind: Secret
metadata:
  name: api-key-secret
  namespace: secrets-lab
type: Opaque
```

Decode to verify:
```bash
kubectl get secret api-key-secret -o jsonpath='{.data.api\.key}' | base64 --decode
# Output: my-super-private-key-content
```

---

## Exercise 5 — Declarative Secret

```bash
kubectl apply -f tls-secret.yaml
# Output: secret/tls-secret created

kubectl get secrets
```

**Expected output:**
```
NAME               TYPE     DATA   AGE
api-key-secret     Opaque   1      2m
db-secret          Opaque   2      5m
tls-secret         Opaque   2      5s
```

Verify the values:
```bash
kubectl get secret tls-secret -o jsonpath='{.data.TLS_CERT}' | base64 --decode
# Output: tls-cert-content

kubectl get secret tls-secret -o jsonpath='{.data.TLS_KEY}' | base64 --decode
# Output: tls-key-content
```

---

## Exercise 6 — Env Var injection (single keys)

```bash
kubectl apply -f pod-env-secret.yaml
# Output: pod/pod-env-secret created

kubectl logs pod-env-secret
```

**Expected output:**
```
DB_USER=admin
DB_PASSWORD=supersecret123
```

> The pod receives the **decoded** values as plain env vars. The base64 decoding is handled automatically by the kubelet.

---

## Exercise 7 — envFrom (all keys at once)

```bash
kubectl apply -f pod-envfrom-secret.yaml
# Output: pod/pod-envfrom-secret created

kubectl logs pod-envfrom-secret
```

**Expected output:**
```
DB_PASSWORD=supersecret123
DB_USER=admin
```

> `envFrom` is convenient when you want to inject all keys from a Secret without listing them individually. Useful for apps that read many config values from env.

---

## Exercise 8 — Volume mount

```bash
kubectl apply -f pod-volume-secret.yaml
# Output: pod/pod-volume-secret created

kubectl logs pod-volume-secret
```

**Expected output:**
```
DB_PASSWORD
DB_USER
admin
```

> Line 1 & 2: `ls /etc/db-creds` — each Secret key becomes a **separate file**
> Line 3: `cat /etc/db-creds/DB_USER` — file content is the decoded value

Exec into the pod to explore further:
```bash
kubectl exec -it pod-volume-secret -- sh
ls /etc/db-creds
# DB_PASSWORD  DB_USER

cat /etc/db-creds/DB_PASSWORD
# supersecret123
exit
```

---

## Exercise 9 — Live update of Secret

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=newpassword456 \
  --dry-run=client -o yaml | kubectl apply -f -
# Output: secret/db-secret configured
```

Wait ~60 seconds, then check the volume-mounted pod:

```bash
kubectl exec pod-volume-secret -- cat /etc/db-creds/DB_PASSWORD
# Output: newpassword456   ← automatically updated!
```

Now check the env-var pod — it will NOT update without a restart:

```bash
kubectl exec pod-env-secret -- env | grep DB_PASSWORD
# Output: DB_PASSWORD=supersecret123   ← still old value
```

**Comparison table:**

| Method | Auto-updates on Secret change? |
|--------|-------------------------------|
| `env.valueFrom.secretKeyRef` | ❌ No — pod restart required |
| `envFrom.secretRef` | ❌ No — pod restart required |
| Volume mount | ✅ Yes — kubelet syncs within ~60s |

---

## Exercise 10 — Docker registry Secret

```bash
kubectl create secret docker-registry my-registry-secret \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myuser@example.com
# Output: secret/my-registry-secret created

kubectl get secret my-registry-secret -o jsonpath='{.type}'
# Output: kubernetes.io/dockerconfigjson
```

To use it in a pod for pulling from a private registry:
```yaml
spec:
  imagePullSecrets:
  - name: my-registry-secret
  containers:
  - name: app
    image: registry.example.com/myapp:latest
```

---

## Exercise 11 — Cleanup

```bash
kubectl delete namespace secrets-lab
# Output: namespace "secrets-lab" deleted

kubectl config set-context --current --namespace=default
# Output: Context "..." modified.
```

---

## Challenge Question Answers

**1. `Opaque` vs `kubernetes.io/dockerconfigjson`?**
- `Opaque` is the generic default — arbitrary key-value pairs, no special handling by Kubernetes.
- `kubernetes.io/dockerconfigjson` is a typed secret specifically for registry credentials. Kubernetes knows how to use it automatically with `imagePullSecrets`. Other types include `kubernetes.io/tls`, `kubernetes.io/service-account-token`, etc.

**2. Why base64 and not encrypted?**
- base64 is just encoding for safe storage of binary data in YAML/JSON, not security.
- By default, Secrets are stored in etcd in plain base64. Anyone with etcd access can read them.
- This is a known limitation. Kubernetes provides **Encryption at Rest** (`EncryptionConfiguration`) to encrypt Secret data in etcd using AES or KMS providers.

**3. Encryption at Rest in etcd?**
- Configured via `EncryptionConfiguration` on the API server.
- Supports providers: `aescbc`, `aesgcm`, `secretbox`, `kms` (external KMS like GCP KMS, AWS KMS, HashiCorp Vault).
- On GKE, Application-layer secrets encryption can be enabled via Cloud KMS.

**4. Volume mount vs Env var for Secrets?**

| Scenario | Recommendation |
|----------|----------------|
| Secret needs to auto-update without pod restart | Volume mount |
| App reads config from files (e.g. TLS certs, kubeconfig) | Volume mount |
| App reads from env vars (12-factor app style) | Env var |
| Simplicity for small number of keys | Env var |
| Large number of keys | `envFrom` or volume mount |

**5. Does updating a Secret update env vars in a running pod?**
- **No.** Env vars are set at pod start time and are **static** for the pod's lifetime.
- To pick up new Secret values via env vars, you must **delete and recreate the pod** (or roll the Deployment).
- Volume-mounted secrets **do** update automatically within ~60 seconds (kubelet sync interval).
