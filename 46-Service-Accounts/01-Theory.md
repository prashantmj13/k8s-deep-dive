# 46 — Service Accounts

## What are Service Accounts?

Service Accounts are identities for **processes running inside pods** to authenticate to the Kubernetes API server. While user accounts are for humans, service accounts are for applications.

---

## Key Facts

- Every namespace has a `default` service account
- Every pod is assigned a service account (default if not specified)
- A JWT token is automatically mounted into pods at `/var/run/secrets/kubernetes.io/serviceaccount/token`
- Service accounts are namespaced resources
- By default the default SA has no RBAC permissions (post v1.24)

---

## Automatic Token Mount

Every pod gets these files mounted automatically:

```
/var/run/secrets/kubernetes.io/serviceaccount/
├── token      ← JWT token for API authentication
├── ca.crt     ← cluster CA cert for TLS verification
└── namespace  ← current namespace
```

---

## Creating and Using Service Accounts

```bash
# Create
kubectl create serviceaccount myapp-sa

# View
kubectl get serviceaccounts
kubectl describe serviceaccount myapp-sa

# Use in a pod
```

```yaml
spec:
  serviceAccountName: myapp-sa
  containers:
  - name: app
    image: myapp:latest
```

---

## Disable Token Auto-mount

If your pod doesn't need API access, disable the mount:

```yaml
spec:
  serviceAccountName: myapp-sa
  automountServiceAccountToken: false
  containers:
  - name: app
    image: myapp:latest
```

Or disable at the SA level:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-api-sa
automountServiceAccountToken: false
```

---

## Token Changes: v1.22+ and v1.24+

| Version | Change |
|---------|--------|
| Pre-v1.22 | Tokens were non-expiring Secrets |
| v1.22+ | Tokens are **projected** (bound, expiring, auto-rotated) |
| v1.24+ | SA Secrets no longer auto-created |

In v1.22+, tokens are projected volumes — they expire (default 1 hour) and are automatically refreshed by the kubelet.

---

## Create a Long-lived Token (v1.24+)

```bash
# If you need a non-expiring token (e.g. for external access)
kubectl create token myapp-sa --duration=8760h

# Or create a Secret-based token
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: myapp-sa-token
  annotations:
    kubernetes.io/service-account.name: myapp-sa
type: kubernetes.io/service-account-token
EOF

kubectl get secret myapp-sa-token -o jsonpath='{.data.token}' | base64 -d
```

---

## Granting API Permissions to a Service Account

```bash
# Create a role
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods

# Bind to service account
kubectl create rolebinding myapp-sa-pod-reader \
  --role=pod-reader \
  --serviceaccount=default:myapp-sa
```

In RBAC, service accounts are referenced as:
```
system:serviceaccount:<namespace>:<name>
```

---

## Using a Service Account Token Externally

```bash
# Get the token
TOKEN=$(kubectl create token myapp-sa)

# Use it to call the API
curl -k https://localhost:6443/api/v1/pods \
  -H "Authorization: Bearer $TOKEN"
```

---

## Service Account vs User Account

| Feature | User Account | Service Account |
|---------|-------------|-----------------|
| Managed by | External (certs, OIDC) | Kubernetes |
| Scope | Cluster-wide | Namespaced |
| Used by | Humans, kubectl | Pods, controllers |
| Token | Cert or external token | JWT (auto-mounted) |
| Created with | openssl / OIDC | kubectl create sa |
