# 34 — Security Primitives | Solutions

---

## Exercise 1 — API server settings

```
--authorization-mode=Node,RBAC
--enable-admission-plugins=NodeRestriction
--client-ca-file=/etc/kubernetes/pki/ca.crt
```

---

## Challenge Answers

**1. Three phases before etcd?**
Authentication (AuthN) → Authorization (AuthZ) → Admission Control. Only requests passing all three reach etcd.

**2. RBAC for production?**
RBAC (`--authorization-mode=RBAC`) is recommended because it enforces least-privilege access — you explicitly grant permissions rather than writing complex policies. It integrates natively with namespaces and service accounts.

**3. Mutating vs Validating admission controller?**
- Mutating: can modify the request object (e.g., inject a sidecar, set default values)
- Validating: can only approve or reject — cannot modify
- Mutating runs first (to ensure validating sees the final object)

**4. Why TLS everywhere?**
Without TLS, any component on the network could intercept, read, or tamper with cluster traffic. etcd contains all secrets, configs, and cluster state — plaintext communication would expose everything to eavesdropping or man-in-the-middle attacks.

**5. Node,RBAC meaning?**
Multiple authorization modes are evaluated in order. A request is allowed if ANY mode approves it. `Node` mode gives kubelets special permissions to manage their own pods, nodes, and related objects. `RBAC` handles all other access control. Using both ensures kubelets work correctly while all other access is governed by RBAC.
