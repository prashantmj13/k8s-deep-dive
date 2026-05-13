# 46 — Service Accounts | Solutions

---

## Exercise 3 — SA volume files

```bash
kubectl exec pod-with-sa -- ls /var/run/secrets/kubernetes.io/serviceaccount/
# ca.crt  namespace  token
```

---

## Exercise 4 — Decoded token payload

```json
{
  "aud": ["https://kubernetes.default.svc.cluster.local"],
  "exp": 1747526400,
  "iat": 1747440000,
  "iss": "https://kubernetes.default.svc.cluster.local",
  "kubernetes.io": {
    "namespace": "sa-lab",
    "pod": {"name": "pod-with-sa", "uid": "abc-123"},
    "serviceaccount": {"name": "dashboard-sa", "uid": "def-456"}
  },
  "sub": "system:serviceaccount:sa-lab:dashboard-sa"
}
```

The `sub` field is `system:serviceaccount:sa-lab:dashboard-sa` — exactly how RBAC identifies this SA.

---

## Exercise 7 — No token mount

```bash
kubectl exec pod-no-token -- ls /var/run/secrets/
# ls: /var/run/secrets/: No such file or directory
```

The secrets volume is not mounted at all when `automountServiceAccountToken: false`.

---

## Challenge Answers

**1. Three auto-mounted files?**
- `token` — JWT token for authenticating to the API server
- `ca.crt` — cluster CA cert, so the pod can verify the API server's TLS cert
- `namespace` — current namespace (so the app knows where it's running)

**2. Prevent API access?**
Set `automountServiceAccountToken: false` on the pod spec (or on the ServiceAccount object itself). This skips the volume mount entirely. Also ensure the SA has no RBAC permissions granted.

**3. v1.24 SA token changes?**
Before v1.24: creating a ServiceAccount automatically created a Secret containing a non-expiring token. After v1.24: no Secret is auto-created. Tokens are projected volumes (bound to pod lifetime, auto-expire/refresh). To get a long-lived token: use `kubectl create token <sa>` or create a `kubernetes.io/service-account-token` Secret manually.

**4. RBAC subject reference?**
```yaml
subjects:
- kind: ServiceAccount
  name: my-sa
  namespace: my-namespace
```
Or as a user string: `system:serviceaccount:<namespace>:<name>`

**5. Share SA across namespaces?**
No. Service accounts are namespaced. A pod in `namespace-a` can only use SAs from `namespace-a`. If two teams need the same permissions, create identical SAs in each namespace or use ClusterRoleBindings that reference SAs by their full path.
