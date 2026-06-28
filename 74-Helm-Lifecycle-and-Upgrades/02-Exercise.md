# 74 — Helm Lifecycle and Upgrades | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 30 minutes

---

## Exercise 1 — Install and create multiple revisions

```bash
kubectl create namespace helm-life
helm repo add bitnami https://charts.bitnami.com/bitnami

helm install myapp bitnami/nginx -n helm-life
helm upgrade myapp bitnami/nginx --set replicaCount=3 -n helm-life
helm upgrade myapp bitnami/nginx --set replicaCount=5 -n helm-life

helm history myapp -n helm-life
```

---

## Exercise 2 — Rollback to a previous revision

```bash
helm rollback myapp 1 -n helm-life
helm history myapp -n helm-life
helm get values myapp -n helm-life
kubectl get deployment -n helm-life
```

> **Observe:** A NEW revision (4) is created that matches revision 1's config — replicaCount should be back to default.

---

## Exercise 3 — Use upgrade --install (idempotent)

```bash
helm uninstall myapp -n helm-life

# This single command works whether or not myapp already exists
helm upgrade --install myapp bitnami/nginx -n helm-life
helm upgrade --install myapp bitnami/nginx --set replicaCount=2 -n helm-life
helm list -n helm-life
```

---

## Exercise 4 — Test --atomic auto-rollback

```bash
helm upgrade myapp bitnami/nginx --set image.tag=does-not-exist-xyz --atomic --timeout 90s -n helm-life
helm history myapp -n helm-life
helm get values myapp -n helm-life
```

> **Observe:** The upgrade fails (image pull error), and `--atomic` triggers an automatic rollback — the release returns to its previous working state.

---

## Exercise 5 — Add a post-install hook

```yaml
# templates/post-install-job.yaml (add to a local chart)
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-post-install"
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: notify
        image: busybox
        command: ["echo", "Post-install hook ran!"]
      restartPolicy: Never
```

```bash
helm create hookchart
cp post-install-job.yaml hookchart/templates/
helm install hooktest ./hookchart -n helm-life
kubectl get jobs -n helm-life
kubectl logs job/hooktest-post-install -n helm-life
```

---

## Exercise 6 — Run helm test

```bash
helm test myapp -n helm-life 2>&1 || echo "no tests defined in bitnami chart by default"
```

---

## Exercise 7 — Uninstall with and without history

```bash
helm uninstall hooktest -n helm-life --keep-history
helm history hooktest -n helm-life     # still works

helm uninstall myapp -n helm-life
helm history myapp -n helm-life        # should fail — no history kept
```

---

## Exercise 8 — Cleanup

```bash
kubectl delete namespace helm-life
rm -rf hookchart
```

---

## Challenge Questions

1. Does `helm rollback` delete the revisions that came after the one you rolled back to?
2. What's the practical benefit of `helm upgrade --install` in a CI/CD pipeline?
3. What does `--atomic` do differently from a plain `helm upgrade`?
4. What is the purpose of `helm.sh/hook-delete-policy`?
5. Why would `--keep-history` matter for compliance/auditing purposes?

---

# Solutions

**Exercise 1 output:**
```
REVISION  STATUS      DESCRIPTION
1         superseded  Install complete
2         superseded  Upgrade complete
3         deployed    Upgrade complete
```

**Exercise 2 output:**
```
REVISION  STATUS      DESCRIPTION
1         superseded  Install complete
2         superseded  Upgrade complete
3         superseded  Upgrade complete
4         deployed    Rollback to 1
```

**Exercise 4:** History shows a revision marked `failed` followed immediately by an auto-generated rollback revision restoring the prior working state.

## Challenge Answers

**1. Does rollback delete later revisions?**
No. Rollback ADDS a new revision at the end of history that re-applies an earlier revision's manifest/values — it never deletes anything. Your full audit trail (1, 2, 3, then 4=rollback-to-1) remains intact.

**2. upgrade --install in CI/CD?**
It makes deployment scripts idempotent and environment-agnostic — the same command works for the very first deploy AND every subsequent deploy, without needing conditional logic to check "does this release already exist?" This greatly simplifies pipeline scripts.

**3. --atomic vs plain upgrade?**
A plain `helm upgrade` that fails leaves the release in a `failed` state with partially-applied resources — manual intervention needed. `--atomic` automatically triggers a rollback to the last successful revision if the upgrade fails or times out, self-healing without operator action.

**4. hook-delete-policy purpose?**
Controls when Helm cleans up hook resources (like Jobs). `hook-succeeded` deletes the Job only if it completed successfully (useful for debugging failures — the failed Job's pod/logs stay around). `before-hook-creation` deletes any leftover Job from a previous release before creating a new one (prevents naming conflicts on retries).

**5. --keep-history for compliance?**
Without it, `helm uninstall` removes all release Secrets — losing the entire revision history and audit trail of what was ever deployed, with what config, and when. For regulated environments needing deployment audit trails, `--keep-history` preserves this record even after the release is removed from the live cluster.
