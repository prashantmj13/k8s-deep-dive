# 76 — Kustomize: Installation, Setup, and kustomization.yaml

## Installation

```bash
# kubectl has kustomize built in (slightly older version)
kubectl version --client

# Install the standalone, up-to-date kustomize binary
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/

# Verify
kustomize version

# macOS
brew install kustomize
```

---

## The kustomization.yaml File

This is the single config file that drives everything — it must be named exactly `kustomization.yaml` (or `.yml`, or `Kustomization`) and live in the directory you point kustomize at.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Which raw manifest files or other kustomizations to include
resources:
- deployment.yaml
- service.yaml
- ../../base          # can reference another kustomization directory

# Apply these labels to ALL generated resources
commonLabels:
  app: myapp
  managed-by: kustomize

# Apply these annotations to ALL generated resources
commonAnnotations:
  team: platform

# Prefix/suffix all resource names
namePrefix: prod-
nameSuffix: -v2

# Set/override the namespace for all resources
namespace: production

# Override container images
images:
- name: myapp
  newName: myregistry.io/myapp
  newTag: v2.5.0

# Generate ConfigMaps/Secrets from literals or files
configMapGenerator:
- name: app-config
  literals:
  - LOG_LEVEL=info
  - ENV=production

secretGenerator:
- name: app-secret
  literals:
  - API_KEY=abc123

# Patches to modify specific fields
patches:
- target:
    kind: Deployment
    name: myapp
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
```

---

## Running Kustomize

```bash
# Render only (no apply) — ALWAYS do this first to review output
kustomize build overlays/prod/
kubectl kustomize overlays/prod/        # equivalent, using kubectl's built-in

# Apply directly
kubectl apply -k overlays/prod/

# Diff before applying (if kubectl diff is available)
kubectl diff -k overlays/prod/

# Delete resources defined by a kustomization
kubectl delete -k overlays/prod/
```

---

## Minimal Working Example

```bash
mkdir -p demo && cd demo
cat > deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
spec:
  replicas: 1
  selector:
    matchLabels: {app: demo}
  template:
    metadata:
      labels: {app: demo}
    spec:
      containers:
      - name: demo
        image: nginx
EOF

cat > kustomization.yaml <<EOF
resources:
- deployment.yaml
commonLabels:
  env: dev
EOF

kustomize build .
```

Output includes the deployment with `env: dev` added to labels — without ever touching `deployment.yaml` directly.

---

## kustomization.yaml Top-Level Field Reference

| Field | Purpose |
|-------|---------|
| `resources` | Files or directories to include |
| `bases` | (deprecated, use `resources`) other kustomizations to build on |
| `components` | Reusable, optional building blocks (see topic 78) |
| `patches` | Strategic-merge or JSON6902 patches |
| `patchesStrategicMerge` | (legacy field name, now just use `patches`) |
| `images` | Override image name/tag/digest |
| `replicas` | Set replica count for named resources |
| `commonLabels` | Labels added everywhere + used in selectors |
| `commonAnnotations` | Annotations added everywhere |
| `namePrefix` / `nameSuffix` | Rename all resources |
| `namespace` | Set/override namespace |
| `configMapGenerator` / `secretGenerator` | Generate Config/Secret objects |
| `vars` (legacy) / `replacements` (modern) | Cross-reference values between resources |
| `generatorOptions` | Control hash suffixes on generated ConfigMaps/Secrets |

---

## Generator Hash Suffixes

By default, generated ConfigMaps/Secrets get a content hash suffix (`app-config-a1b2c3d4`), and any Deployment referencing them is automatically updated to point at the new name — this is how Kustomize triggers rolling restarts on config changes without manual versioning.

```yaml
generatorOptions:
  disableNameSuffixHash: false   # keep hash suffix (recommended, default)
```

---

## Checking and Validating

```bash
kustomize build . > /tmp/rendered.yaml
kubectl apply --dry-run=client -f /tmp/rendered.yaml
kubectl apply --dry-run=server -f /tmp/rendered.yaml   # validates against live API schema
```
