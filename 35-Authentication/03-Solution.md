# 35 — Authentication | Solutions

---

## Exercise 5 — Certificate inspection

```bash
cat /tmp/jane-signed.crt | openssl x509 -text | grep -E "Subject:|Issuer:"
```
```
Issuer: CN = kubernetes
Subject: CN = jane, O = developers
```

The CN `jane` becomes the Kubernetes username. The O `developers` becomes the group.

---

## Exercise 6 — Forbidden access

```bash
kubectl get pods
```
```
Error from server (Forbidden): pods is forbidden: User "jane" cannot list resource "pods" in API group "" in the namespace "default"
```

Authentication succeeded (jane's cert is valid) but authorization failed (no RBAC role assigned). We'll fix this in the RBAC topic.

---

## Exercise 7 — Service account token

```bash
kubectl exec sa-test -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
# Output: eyJhbGciOiJSUzI1NiIsImtpZCI6Ii...  (JWT token)
```

---

## Challenge Answers

**1. CN field?**
The `CN` (Common Name) field in a client certificate becomes the **Kubernetes username**. When the API server receives a request, it extracts CN to identify who is making the request.

**2. O field?**
The `O` (Organization) field becomes the **group membership**. A user can be in multiple groups. Groups are used in RBAC to assign permissions to multiple users at once (e.g., `O=system:masters` gives cluster-admin access).

**3. Can you create a Kubernetes user with kubectl?**
No. Kubernetes has no built-in user management. Users exist only as certificate CN values or external identity tokens. There's no `kubectl create user` command. You manage users by issuing/revoking TLS certificates or managing external identity providers.

**4. Default kubeconfig location?**
`~/.kube/config`. Override with the `KUBECONFIG` environment variable or `--kubeconfig` flag.

**5. Pod without serviceAccountName?**
The `default` service account in the pod's namespace is automatically assigned. Its token is still mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`. The default SA typically has minimal permissions unless explicitly granted RBAC roles.
