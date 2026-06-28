# 78 — Kustomize Patches, Overlays, Components | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 30 minutes

---

## Exercise 1 — Base setup

```bash
mkdir -p app/base app/overlays/prod app/components/limits

cat > app/base/deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels: {app: myapp}
  template:
    metadata:
      labels: {app: myapp}
    spec:
      containers:
      - name: myapp
        image: nginx:1.25
        env:
        - name: LOG_LEVEL
          value: "info"
EOF

cat > app/base/kustomization.yaml <<EOF
resources:
- deployment.yaml
EOF
```

---

## Exercise 2 — Strategic merge patch (separate file)

```bash
cat > app/overlays/prod/replica-patch.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 6
EOF

cat > app/overlays/prod/kustomization.yaml <<EOF
resources:
- ../../base
patches:
- path: replica-patch.yaml
  target:
    kind: Deployment
    name: myapp
EOF

kustomize build app/overlays/prod/ | grep replicas
```

---

## Exercise 3 — JSON6902 patch to add an env var

```bash
cat >> app/overlays/prod/kustomization.yaml <<EOF
- target:
    kind: Deployment
    name: myapp
  patch: |-
    - op: add
      path: /spec/template/spec/containers/0/env/-
      value: {name: ENV, value: "production"}
    - op: replace
      path: /spec/template/spec/containers/0/env/0/value
      value: "warn"
EOF

kustomize build app/overlays/prod/ | grep -A6 "env:"
```

---

## Exercise 4 — Create a reusable Component

```bash
cat > app/components/limits/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
patches:
- target:
    kind: Deployment
  patch: |-
    - op: add
      path: /spec/template/spec/containers/0/resources
      value:
        requests: {cpu: "200m", memory: "256Mi"}
        limits: {cpu: "500m", memory: "512Mi"}
EOF
```

---

## Exercise 5 — Use the component in the overlay

```bash
cat >> app/overlays/prod/kustomization.yaml <<EOF
components:
- ../../components/limits
EOF

kustomize build app/overlays/prod/ | grep -A6 "resources:"
```

---

## Exercise 6 — Apply and verify

```bash
kubectl apply -k app/overlays/prod/
kubectl get deployment myapp -o jsonpath='{.spec.replicas}'
kubectl get deployment myapp -o jsonpath='{.spec.template.spec.containers[0].resources}'
kubectl get deployment myapp -o jsonpath='{.spec.template.spec.containers[0].env}'
```

---

## Exercise 7 — Diff base vs overlay output

```bash
kustomize build app/base/ > /tmp/base.yaml
kustomize build app/overlays/prod/ > /tmp/prod.yaml
diff /tmp/base.yaml /tmp/prod.yaml
```

---

## Exercise 8 — Cleanup

```bash
kubectl delete -k app/overlays/prod/
rm -rf app /tmp/base.yaml /tmp/prod.yaml
```

---

## Challenge Questions

1. When would you choose a JSON6902 patch over a strategic merge patch?
2. What's the key structural difference between a `Kustomization` and a `Component` file?
3. Why are components useful when you have many overlays that need the same cross-cutting change?
4. What happens if a JSON6902 `replace` operation targets a path that doesn't exist?
5. How would you verify exactly what an overlay changes relative to its base, before applying?

---

# Solutions

**Exercise 2:** `replicas: 6` in the rendered output.

**Exercise 3:**
```yaml
env:
- name: LOG_LEVEL
  value: warn
- name: ENV
  value: production
```

**Exercise 5:**
```yaml
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 200m
    memory: 256Mi
```

## Challenge Answers

**1. JSON6902 vs strategic merge?**
Use JSON6902 when you need precise array element manipulation (insert at a specific index, remove a specific element, append with `/-`) or when you want to be explicit/unambiguous about exactly which operation is happening. Strategic merge is more natural for merging whole sub-objects (like adding several fields to a container spec at once) since it reads like normal YAML.

**2. Kustomization vs Component structural difference?**
A `Kustomization` (kind: Kustomization) is a complete, standalone build unit — it can include `resources` (actual objects) and is the top-level thing you point `kustomize build` at. A `Component` (kind: Component) typically contains only `patches`/transformers with NO `resources` of its own — it's meant to be referenced FROM a Kustomization's `components:` list, not built directly.

**3. Why components for cross-cutting changes?**
Without components, adding (say) a security patch or sidecar injection annotation to 10 different overlays means copy-pasting the same patch into all 10 `kustomization.yaml` files — any update means editing 10 places. A component centralizes that patch once; every overlay that opts in via `components: [../../components/x]` automatically gets updates when the component changes.

**4. JSON6902 replace on nonexistent path?**
The patch fails with an error (something like "path does not exist") — `replace` requires the target path to already exist (unlike `add`, which can create it). This is a meaningful safety difference: `add` is more forgiving for new fields, `replace` is stricter and catches typos/wrong assumptions about base structure.

**5. Verify overlay's exact changes?**
Render both `kustomize build base/` and `kustomize build overlay/` to files, then run a plain `diff` between them — this shows precisely which fields the overlay adds, changes, or removes relative to the base, with zero cluster interaction required.
