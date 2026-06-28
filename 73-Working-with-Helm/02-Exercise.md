# 73 — Working with Helm | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 25 minutes

---

## Exercise 1 — Install with --set overrides

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
kubectl create namespace helm-work

helm install web bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=ClusterIP \
  -n helm-work

helm get values web -n helm-work
kubectl get deployment -n helm-work
```

---

## Exercise 2 — Install with a custom values file

```yaml
# custom.yaml
replicaCount: 2
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi
```

```bash
helm upgrade web bitnami/nginx -f custom.yaml -n helm-work
helm get values web -n helm-work
```

---

## Exercise 3 — Test precedence: -f vs --set

```bash
helm upgrade web bitnami/nginx -f custom.yaml --set replicaCount=7 -n helm-work
helm get values web -n helm-work
```

> **Observe:** replicaCount is 7, not 2 — `--set` wins over `-f`.

---

## Exercise 4 — Use --dry-run --debug to catch errors

```bash
helm install broken bitnami/nginx --set replicaCount=notanumber --dry-run --debug -n helm-work
```

> **Observe:** Should fail validation before touching the cluster.

---

## Exercise 5 — View all effective values including defaults

```bash
helm get values web -n helm-work          # only user-supplied overrides
helm get values web -n helm-work --all    # full merged values
```

---

## Exercise 6 — Use --atomic for safe upgrades

```bash
# This should succeed
helm upgrade web bitnami/nginx --set replicaCount=2 --atomic --timeout 2m -n helm-work

# Simulate a bad upgrade that should auto-rollback
helm upgrade web bitnami/nginx --set image.tag=this-tag-does-not-exist --atomic --timeout 1m -n helm-work
helm history web -n helm-work
```

---

## Exercise 7 — Cleanup

```bash
helm uninstall web -n helm-work
kubectl delete namespace helm-work
```

---

## Challenge Questions

1. If you pass both `-f values.yaml` and `--set key=value` for the same key, which wins?
2. What does `--dry-run --debug` actually validate?
3. What happens when you run `helm upgrade` with `--atomic` and the upgrade fails?
4. Why would you use `--set-string` instead of `--set` for a version number like "1.0"?
5. What command shows ONLY what you explicitly overrode, vs the full merged config?

---

# Solutions

**Exercise 3:** `replicaCount: 7` — `--set` always has the highest precedence, overriding both the values file and chart defaults.

**Exercise 4:** Helm fails with a type validation error before any cluster call is made — `--dry-run --debug` renders and schema-checks templates locally.

**Exercise 6:** `helm history web -n helm-work` shows a revision marked as rolled back automatically after the failed upgrade — `--atomic` triggers an automatic `helm rollback` if the upgrade doesn't succeed within the timeout.

## Challenge Answers

**1. -f vs --set precedence?**
`--set` wins. Precedence (lowest to highest): chart's `values.yaml` defaults → `-f` files (later files override earlier ones) → `--set`/`--set-string`/`--set-file` flags.

**2. What --dry-run --debug validates?**
It renders all templates with the computed/merged values and submits them to the API server in dry-run mode (server-side validation against the live cluster's API schema) without persisting anything. It catches both template syntax errors and Kubernetes object schema violations (e.g. invalid types, missing required fields) before a real install.

**3. --atomic on failure?**
If the upgrade fails (timeout, hook failure, or resource error), Helm automatically performs a rollback to the previous successful revision — leaving the cluster in the last known-good state instead of a half-applied broken upgrade.

**4. Why --set-string for "1.0"?**
Without it, Helm's `--set` parser may interpret `1.0` as a YAML float and coerce it, potentially losing the trailing zero or converting types unexpectedly (e.g., `1.0` becomes `1`). `--set-string` forces the value to remain a literal string exactly as typed.

**5. Show only overrides vs full config?**
`helm get values <release>` shows only what you explicitly supplied (the override layer). `helm get values <release> --all` shows the fully merged configuration including all chart defaults that weren't overridden.
