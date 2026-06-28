# 73 — Working with Helm: Customizing Chart Parameters and Deploying

## Three Ways to Override Values

### 1. --set (inline, quick overrides)

```bash
helm install myapp bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=LoadBalancer \
  --set image.tag=1.25.3

# Nested/array values
helm install myapp ./mychart \
  --set ingress.hosts[0].host=app.example.com \
  --set resources.limits.cpu=500m
```

### 2. -f / --values (custom values file)

```yaml
# custom-values.yaml
replicaCount: 5
image:
  tag: "2.0.0"
service:
  type: LoadBalancer
resources:
  requests:
    cpu: 200m
    memory: 256Mi
```

```bash
helm install myapp ./mychart -f custom-values.yaml

# Layer multiple files — later files override earlier ones
helm install myapp ./mychart -f base-values.yaml -f prod-values.yaml
```

### 3. --set-string / --set-file (special cases)

```bash
# Force a value to be treated as a string (avoid numeric/bool coercion)
helm install myapp ./mychart --set-string version="1.0"

# Load a value from a file's contents
helm install myapp ./mychart --set-file config=app.conf
```

---

## Precedence Order (highest wins)

```
--set (and --set-string/--set-file)     ← highest priority
        ↓
-f custom-values.yaml (last file wins if multiple -f flags)
        ↓
values.yaml (chart defaults)             ← lowest priority
```

---

## Inspecting Effective Values

```bash
# See merged values for an installed release
helm get values myapp

# Include chart defaults too
helm get values myapp --all

# Preview merged values without installing
helm template ./mychart -f custom-values.yaml --debug
```

---

## Deploying a Chart — Full Workflow

```bash
# 1. Search and inspect
helm search repo bitnami/postgresql
helm show values bitnami/postgresql > defaults.yaml

# 2. Customize
cat > my-postgres-values.yaml <<EOF
auth:
  postgresPassword: "supersecret"
primary:
  persistence:
    size: 10Gi
resources:
  requests:
    cpu: 250m
    memory: 256Mi
EOF

# 3. Dry-run to catch errors before applying
helm install my-db bitnami/postgresql -f my-postgres-values.yaml --dry-run --debug

# 4. Actually install
helm install my-db bitnami/postgresql -f my-postgres-values.yaml -n database --create-namespace

# 5. Verify
helm status my-db -n database
kubectl get pods -n database
```

---

## --dry-run and --debug

```bash
helm install myapp ./mychart --dry-run --debug
```

This renders the FULL chart (templates + computed values) and validates against the API server schema WITHOUT actually creating resources — the single most useful command for catching template bugs before they hit your cluster.

---

## Setting Namespace and Creating it Automatically

```bash
helm install myapp ./mychart -n myns --create-namespace
```

---

## Generating a Release Name Automatically

```bash
helm install ./mychart --generate-name
# Generates something like: mychart-1620000000
```

---

## Useful Flags Summary

| Flag | Purpose |
|------|---------|
| `-f FILE` | Use a values file (repeatable, later wins) |
| `--set k=v` | Inline override (repeatable) |
| `--dry-run` | Render and validate without applying |
| `--debug` | Verbose output, shows computed values |
| `-n NAMESPACE` | Target namespace |
| `--create-namespace` | Create the namespace if missing |
| `--wait` | Wait until all resources are ready before returning |
| `--timeout 5m` | Max wait time for `--wait` or hooks |
| `--atomic` | Roll back automatically if install/upgrade fails |

---

## Production-grade Install Example

```bash
helm install myapp ./mychart \
  -f values-prod.yaml \
  -n production \
  --create-namespace \
  --atomic \
  --timeout 10m \
  --wait
```
