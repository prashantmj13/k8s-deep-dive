# 77 — Kustomize: API/Kind, Managing Directories, Transformers, Image Transformation

## The Kustomization API and Kind

Every `kustomization.yaml` declares its own API version and kind, just like any Kubernetes resource — but it is processed entirely client-side by the kustomize tool, never sent to the API server as-is:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
```

This is the ONLY valid `kind` for this file — there's no other "kind" of kustomization object.

---

## Managing Directories: Bases and Overlay Layout

### Recommended directory structure

```
myapp/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml      # resources: [deployment.yaml, service.yaml, configmap.yaml]
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml   # resources: [../../base]
    │   └── replica-patch.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── prod/
        ├── kustomization.yaml
        └── resource-limits-patch.yaml
```

### base/kustomization.yaml
```yaml
resources:
- deployment.yaml
- service.yaml
- configmap.yaml
```

### overlays/prod/kustomization.yaml
```yaml
resources:
- ../../base
namePrefix: prod-
commonLabels:
  env: prod
patches:
- path: resource-limits-patch.yaml
images:
- name: myapp
  newTag: v3.0.0
```

```bash
kubectl apply -k overlays/prod/
kubectl apply -k overlays/dev/
```

### Referencing Remote Bases (Git)

```yaml
resources:
- github.com/kubernetes-sigs/kustomize/examples/multibases?ref=v4.5.7
```

Kustomize can pull bases directly from a Git URL — useful for sharing a common base across multiple repos.

---

## Common Transformers

Transformers are built-in operations applied to ALL resources in a kustomization without per-resource patches.

### namePrefix / nameSuffix
```yaml
namePrefix: prod-
nameSuffix: -v1
```

### commonLabels (also updates selectors!)
```yaml
commonLabels:
  app.kubernetes.io/managed-by: kustomize
```
> Unlike a simple find/replace, `commonLabels` updates BOTH the metadata labels AND any matching `selector` fields (Deployment, Service) so they stay consistent.

### commonAnnotations
```yaml
commonAnnotations:
  team: platform
  contact: platform@example.com
```

### namespace
```yaml
namespace: production
```

### replicas (transformer, simpler than a patch)
```yaml
replicas:
- name: myapp
  count: 5
```

### labels (newer, more flexible than commonLabels)
```yaml
labels:
- pairs:
    env: prod
  includeSelectors: true
  includeTemplates: true
```

---

## Image Transformation

The most commonly used transformer — swap image name/tag/digest without touching the Deployment YAML at all:

```yaml
images:
- name: myapp                          # matches the image name used in the base manifest
  newName: myregistry.io/myapp         # optional: change the registry/repo
  newTag: v2.5.0                       # change the tag
- name: nginx
  digest: sha256:abc123...             # pin to a digest instead of a tag
```

**Base manifest (unchanged):**
```yaml
containers:
- name: app
  image: myapp:latest
```

**After kustomize build with the images transformer above:**
```yaml
containers:
- name: app
  image: myregistry.io/myapp:v2.5.0
```

This is the standard CI/CD pattern: the base always says `:latest` or a placeholder, and the CI pipeline injects the actual built tag via `kustomize edit set image`:

```bash
cd overlays/prod
kustomize edit set image myapp=myregistry.io/myapp:v2.5.0
kustomize build .
```

---

## Combining Multiple Transformers

```yaml
resources:
- ../../base
namePrefix: prod-
namespace: production
commonLabels:
  env: prod
images:
- name: myapp
  newTag: v3.0.0
replicas:
- name: myapp
  count: 8
```

`kustomize build` applies generators first, then transformers, then patches — in a well-defined, predictable order.

---

## Listing What a Kustomization Will Produce

```bash
kustomize build overlays/prod/ | grep -E "kind:|name:|image:|replicas:"
```

---

## Validating Cross-Directory References

```bash
# Make sure relative paths resolve correctly
cd overlays/prod && kustomize build . 
# common error: "rawResources failed to read Resources: ../../base: evalsymlink failure"
# fix: check the relative path is correct from the overlay's own directory
```
