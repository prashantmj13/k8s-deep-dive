# 51 — CRD | Solutions

---

## Exercise 2 — Custom resource output

```bash
kubectl get appconfigs
```
```
NAME     REPLICAS   ENVIRONMENT   AGE
my-app   3          production    10s
```

---

## Exercise 3 — Validation errors

```
error: AppConfig.stable.example.com "bad-app" is invalid:
  [spec.replicas: Invalid value: 50: spec.replicas in body should be less than or equal to 10,
   spec.environment: Unsupported value: "unknown": supported values: "dev", "staging", "production"]
```

---

## Exercise 5 — After deleting CRD

```bash
kubectl get appconfigs
Error from server (NotFound): Unable to list "stable.example.com/v1, Resource=appconfigs":
the server could not find the requested resource
```

All instances are also deleted automatically when the CRD is deleted.

---

## Challenge Answers

**1. CRD naming convention?**
`<plural>.<group>` — e.g. `appconfigs.stable.example.com`. This must match exactly: the plural form of the resource name, followed by a dot, followed by the API group.

**2. Namespaced vs Cluster scope?**
- `Namespaced`: CRs live in a namespace (`kubectl get appconfigs -n myns`). Affected by RBAC namespace scoping.
- `Cluster`: CRs are cluster-wide, like nodes or PVs (`kubectl get appconfigs` without `-n`). Require ClusterRole for access.

**3. Deleting the CRD?**
All Custom Resource instances of that CRD are **permanently deleted** along with the CRD. This is destructive. Always back up your CRs before deleting a CRD.

**4. served vs storage?**
- `served: true`: This version is available via the API (can be used in kubectl/API calls).
- `storage: true`: This version is used to store the object in etcd. Exactly ONE version can have `storage: true`. Multiple versions can have `served: true` for backward compatibility.

**5. Short names?**
Set in `spec.names.shortNames`:
```yaml
names:
  shortNames: ["ac", "appconf"]
```
Then `kubectl get ac` works as an alias for `kubectl get appconfigs`.
