# 74 — Helm Lifecycle Management and Upgrading Charts

## Release Lifecycle Commands

```bash
helm install   myapp ./chart         # create a new release
helm upgrade   myapp ./chart         # update an existing release
helm rollback  myapp 2               # revert to a specific revision
helm uninstall myapp                 # remove a release
helm history   myapp                 # list all revisions
helm status    myapp                 # show current release status
```

---

## Every Change Creates a New Revision

```bash
helm install myapp ./chart           # revision 1
helm upgrade myapp ./chart --set replicaCount=5    # revision 2
helm upgrade myapp ./chart --set image.tag=2.0     # revision 3

helm history myapp
```
```
REVISION  UPDATED                   STATUS      CHART          APP VERSION  DESCRIPTION
1         Mon May 13 10:00:00 2026  superseded  myapp-1.0.0    1.0.0        Install complete
2         Mon May 13 10:05:00 2026  superseded  myapp-1.0.0    1.0.0        Upgrade complete
3         Mon May 13 10:10:00 2026  deployed    myapp-1.0.0    2.0.0        Upgrade complete
```

---

## upgrade --install (idempotent deploy)

Useful in CI/CD — installs if not present, upgrades if it is:

```bash
helm upgrade --install myapp ./chart -f values-prod.yaml -n production --create-namespace
```

---

## Rollback

```bash
# Roll back to the previous revision
helm rollback myapp

# Roll back to a specific revision
helm rollback myapp 2

# Verify
helm history myapp
helm status myapp
```

Rollback creates a NEW revision (e.g. revision 4) that re-applies the manifests from revision 2 — it does not delete history, it adds to it.

---

## helm diff (plugin) — Preview Before Upgrade

```bash
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade myapp ./chart -f new-values.yaml
```

Shows a kubectl-diff-style preview of exactly what would change — critical for safe production upgrades.

---

## Upgrade Strategies

### Standard upgrade
```bash
helm upgrade myapp ./chart -f values.yaml
```

### Safe upgrade with auto-rollback on failure
```bash
helm upgrade myapp ./chart -f values.yaml --atomic --timeout 5m
```

### Force resource recreation (delete + recreate instead of patch)
```bash
helm upgrade myapp ./chart --force
```
> Use `--force` carefully — it can cause downtime since resources are deleted before being recreated.

### Reuse previously set values not in the new command
```bash
helm upgrade myapp ./chart --reuse-values --set image.tag=2.1
```

---

## Hooks — Lifecycle Event Actions

Helm hooks let you run Jobs at specific points in the release lifecycle:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "1"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: myapp:latest
        command: ["./migrate.sh"]
      restartPolicy: Never
```

| Hook | Fires |
|------|-------|
| `pre-install` | Before templates are installed |
| `post-install` | After all resources are deployed |
| `pre-upgrade` | Before upgrade templates are applied |
| `post-upgrade` | After upgrade completes |
| `pre-rollback` | Before a rollback |
| `post-rollback` | After a rollback |
| `pre-delete` | Before deletion |
| `post-delete` | After deletion |

---

## Uninstalling

```bash
# Standard uninstall
helm uninstall myapp

# Keep history (don't fully purge) — Helm 3 default is to NOT keep history unless --keep-history
helm uninstall myapp --keep-history
helm history myapp     # still works if --keep-history was used

# Dry run uninstall
helm uninstall myapp --dry-run
```

---

## Managing Multiple Environments

```bash
# Same chart, different value overlays per environment
helm upgrade --install myapp ./chart -f values-base.yaml -f values-dev.yaml -n dev
helm upgrade --install myapp ./chart -f values-base.yaml -f values-staging.yaml -n staging
helm upgrade --install myapp ./chart -f values-base.yaml -f values-prod.yaml -n production
```

---

## Helm Test

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-test-connection"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
  - name: wget
    image: busybox
    command: ['wget']
    args: ['{{ .Release.Name }}:80']
  restartPolicy: Never
```

```bash
helm test myapp
```

---

## Full CI/CD Lifecycle Example

```bash
# CI pipeline pseudo-script
helm lint ./chart
helm template ./chart -f values-prod.yaml --debug > /dev/null   # syntax+schema check
helm diff upgrade myapp ./chart -f values-prod.yaml              # preview
helm upgrade --install myapp ./chart -f values-prod.yaml \
  -n production --create-namespace --atomic --timeout 10m --wait
helm test myapp -n production
```
