# 75 — Kustomize: Problem Statement, Philosophy, and vs Helm

## The Problem Kustomize Solves

You have one application that needs slightly different configuration across environments (dev, staging, prod) — different replica counts, image tags, resource limits, ConfigMap values. Two bad options without Kustomize:

1. **Copy-paste YAML per environment** — drift, duplication, hard to maintain
2. **Templating with variables (Helm-style)** — powerful but adds a templating language on top of YAML, can obscure what's actually being applied

---

## Kustomize's Philosophy: "Template-free" Customization

Kustomize takes a different approach: **no templating language at all**. Instead, you keep plain, valid Kubernetes YAML as your "base," and apply structured **patches and transformations** declared in a `kustomization.yaml` file to produce environment-specific variants.

```
base/ (plain valid YAML, no {{ }} syntax anywhere)
   ↓
overlays/dev/      (patches: lower replicas, dev image tag)
overlays/staging/  (patches: staging image tag)
overlays/prod/     (patches: higher replicas, prod resources, prod domain)
```

Every file remains valid, readable Kubernetes YAML on its own — there's no templating syntax to learn, and `kubectl apply -k` (or `kustomize build`) just declaratively recombines/patches existing manifests.

---

## Core Idea: Bases and Overlays

```
- base: the common, shared configuration
- overlay: an environment-specific layer that patches the base
```

```
myapp/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml      # references ../../base + dev patches
    ├── staging/
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

---

## Kustomize vs Helm — Detailed Comparison

| Aspect | Kustomize | Helm |
|--------|-----------|------|
| Templating | None — patches plain YAML | Go templates (`{{ }}` syntax) |
| Learning curve | Lower — it's just YAML + patches | Higher — templating language, functions, pipes |
| Packaging/distribution | No built-in package format | Charts are versioned, packaged (.tgz), pushed to repos |
| Parameterization | Limited (patches, vars, replacements) | Extensive (any value can be a variable) |
| Release tracking | None built-in | Built-in (revisions, rollback, history) |
| Built into kubectl | ✅ Yes (`kubectl apply -k`) | ❌ No — separate binary |
| Best for | Environment overlays of YOUR OWN manifests | Distributing reusable, configurable packages (your apps or 3rd-party software) |
| Community ecosystem | Smaller, mostly DIY | Huge (Artifact Hub, Bitnami, etc.) |

---

## When to Choose Which

**Choose Kustomize when:**
- You own all the YAML and just need per-environment variation
- You want zero extra tooling/binary beyond kubectl
- You want manifests to remain directly readable (no template syntax)
- You're doing GitOps (ArgoCD/Flux both have first-class Kustomize support)

**Choose Helm when:**
- You're distributing a reusable package to others (e.g. publishing a chart)
- You need complex conditional logic, loops, or computed values
- You want built-in release/rollback/versioning semantics
- You're installing third-party software (most vendors publish Helm charts, not Kustomize bases)

**They're not mutually exclusive:** many teams use Helm to install third-party software and Kustomize to manage their own application overlays — or even use Kustomize to post-process rendered Helm output (`helm template | kustomize build`).

---

## No Templating — What You Get Instead

| Kustomize feature | Helm equivalent |
|--------------------|-----------------|
| `patches` (strategic merge / JSON6902) | `{{ if }}` conditionals, value overrides |
| `images` transformer | `--set image.tag=` |
| `replicas` transformer | `--set replicaCount=` |
| `configMapGenerator` / `secretGenerator` | ConfigMap/Secret templates with `.Values` |
| `commonLabels` / `commonAnnotations` | manual labels in every template |
| `components` | subcharts / library charts |
| `vars` / `replacements` | `.Values` cross-references |

---

## Quick Conceptual Example

**base/deployment.yaml** (plain valid YAML):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:latest
```

**overlays/prod/kustomization.yaml**:
```yaml
resources:
- ../../base
patches:
- target:
    kind: Deployment
    name: myapp
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 10
images:
- name: myapp
  newTag: v2.5.0
```

```bash
kubectl apply -k overlays/prod/
```

No `{{ .Values.replicaCount }}` anywhere — the base stays pure, valid YAML; the overlay declares exactly what to change.
