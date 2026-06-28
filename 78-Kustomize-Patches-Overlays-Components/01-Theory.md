# 78 — Kustomize: Patches, Overlays, and Components

## Two Patch Formats

### 1. Strategic Merge Patch (most common, YAML-native)

Looks like a partial Kubernetes resource — Kustomize merges it into the matching base resource by field name:

```yaml
# patches:
- target:
    kind: Deployment
    name: myapp
  patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: myapp
    spec:
      replicas: 5
      template:
        spec:
          containers:
          - name: myapp
            resources:
              limits:
                memory: "512Mi"
```

Or as a separate file:
```yaml
patches:
- path: replica-patch.yaml
  target:
    kind: Deployment
    name: myapp
```

### 2. JSON 6902 Patch (precise, operation-based)

Uses explicit `op`/`path`/`value` — more precise for arrays or when you need to delete/replace specific elements:

```yaml
patches:
- target:
    kind: Deployment
    name: myapp
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
    - op: add
      path: /spec/template/spec/containers/0/env/-
      value: {name: NEW_VAR, value: "hello"}
    - op: remove
      path: /spec/template/spec/containers/0/env/2
```

| Operation | Effect |
|-----------|--------|
| `add` | Insert a value at the path (or append with `/-`) |
| `replace` | Overwrite the value at the path |
| `remove` | Delete the value at the path |

---

## When to Use Which Patch Type

| Use case | Best format |
|----------|-------------|
| Changing a scalar field (replicas, image) | Either — strategic merge is simpler |
| Adding/removing one array element precisely | JSON6902 (`add`/`remove` with index) |
| Merging a partial object structure | Strategic merge (more YAML-natural) |
| Deleting a field entirely | JSON6902 `remove`, or `$patch: delete` in strategic merge |

---

## Patches List — Multiple Patches, Multiple Targets

```yaml
patches:
- path: replica-patch.yaml
  target:
    kind: Deployment
    name: myapp
- path: resource-limits-patch.yaml
  target:
    kind: Deployment
    labelSelector: "tier=backend"
- patch: |-
    - op: replace
      path: /spec/type
      value: LoadBalancer
  target:
    kind: Service
    name: myapp-svc
```

The `target` selector can match by `kind`, `name`, `namespace`, `labelSelector`, or `annotationSelector` — and can match MULTIPLE resources if using a label selector.

---

## Overlay Recap — Layering Patches

```
base/                      ← common config
overlays/
  dev/   → patches: lower replicas, debug logging
  prod/  → patches: higher replicas, resource limits, prod domain
```

Each overlay is independent — applying `overlays/dev/` never touches `overlays/prod/`, both reference the same base read-only.

---

## Components — Reusable, Optional Patch Bundles

Components (kind: `Component`, not `Kustomization`) are reusable chunks of configuration that MULTIPLE overlays can opt into — solving the problem of duplicating the same patch across many overlays.

### components/istio-injection/kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

patches:
- target:
    kind: Deployment
  patch: |-
    - op: add
      path: /spec/template/metadata/annotations/sidecar.istio.io~1inject
      value: "true"
```

### overlays/prod/kustomization.yaml
```yaml
resources:
- ../../base
components:
- ../../components/istio-injection
- ../../components/monitoring
```

Any overlay that lists `../../components/istio-injection` gets that patch applied — without copy-pasting the patch into every overlay's own `patches:` list.

---

## Components vs Bases

| | Base | Component |
|-|------|-----------|
| Kind | `Kustomization` | `Component` |
| Contains | Full resources (Deployments, Services) | Usually just patches/transformers (optional add-ons) |
| Reusability | One base, many overlays extend it | One component, many overlays OPT IN to it |
| Typical use | The "core" of an app | Cross-cutting concerns (sidecar injection, common monitoring, security patches) |

---

## Full Worked Example — Patches + Components Together

```
myapp/
├── base/
│   ├── deployment.yaml
│   └── kustomization.yaml
├── components/
│   └── resource-limits/
│       └── kustomization.yaml   # kind: Component
└── overlays/
    └── prod/
        ├── kustomization.yaml
        └── replica-patch.yaml
```

**overlays/prod/replica-patch.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 8
```

**overlays/prod/kustomization.yaml:**
```yaml
resources:
- ../../base
components:
- ../../components/resource-limits
patches:
- path: replica-patch.yaml
  target:
    kind: Deployment
    name: myapp
```

```bash
kustomize build overlays/prod/
```

---

## Debugging Patches

```bash
# See exactly what a patch changes vs the base alone
kustomize build base/ > /tmp/base-rendered.yaml
kustomize build overlays/prod/ > /tmp/prod-rendered.yaml
diff /tmp/base-rendered.yaml /tmp/prod-rendered.yaml
```
