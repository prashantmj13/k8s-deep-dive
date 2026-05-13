# 43 — Authorization | Solutions

---

## Exercise 2 — auth can-i results

```bash
kubectl auth can-i create pods        # yes (you're cluster-admin)
kubectl auth can-i delete nodes       # yes
kubectl auth can-i "*" "*"            # yes
kubectl auth can-i create pods --as jane   # no
kubectl auth can-i list pods --as jane     # no
kubectl auth can-i get nodes --as jane     # no
```

---

## Exercise 3 — After creating role for jane

```bash
kubectl auth can-i list pods --as jane -n default     # yes
kubectl auth can-i create pods --as jane -n default   # no
kubectl auth can-i list pods --as jane -n kube-system # no
```

---

## Exercise 4 — SubjectAccessReview output

```yaml
status:
  allowed: true
  reason: 'RBAC: allowed by RoleBinding "jane-bind/default" of Role "jane-role/default"
    to User "jane"'
```

---

## Challenge Answers

**1. Node,RBAC meaning?**
Modes are evaluated left to right. A request is allowed if ANY mode permits it. `Node` mode runs first and handles kubelet requests (allowing kubelets to access only their own node's resources). `RBAC` handles all other requests. This combination is the production standard.

**2. Authorization vs Admission Control?**
- Authorization: binary allow/deny based on who the user is and what they want to do. Runs before the request reaches the object validation stage.
- Admission Control: runs after authorization and can validate or mutate the request content. Can reject a request even if the user is authorized (e.g. pod spec violates resource quota, or security context disallows root).

**3. Quick permission check?**
```bash
kubectl auth can-i <verb> <resource> [--as <user>] [-n <namespace>]
```

**4. RBAC deny by default?**
RBAC uses a whitelist model — permissions must be explicitly granted. There is no "deny" rule; the absence of a matching allow rule means denial. This enforces least-privilege: users start with zero permissions and are granted only what they need.

**5. Webhook authorization use case?**
When you need authorization logic that RBAC can't express — for example: time-based access (allow only during business hours), context-aware policies (allow based on request IP or label), or centralized policy management across multiple clusters using a single policy engine like Open Policy Agent (OPA) or Kyverno.
