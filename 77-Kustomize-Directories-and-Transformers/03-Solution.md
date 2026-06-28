# 77 — Kustomize Directories and Transformers | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 30 minutes

---

## Exercise 1 — Build a base + 2 overlays structure

```bash
mkdir -p myapp/base myapp/overlays/dev myapp/overlays/prod

cat > myapp/base/deployment.yaml <<EOF
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
        image: myapp:latest
EOF

cat > myapp/base/kustomization.yaml <<EOF
resources:
- deployment.yaml
EOF
```

---

## Exercise 2 — Dev overlay (low replicas, dev tag)

```bash
cat > myapp/overlays/dev/kustomization.yaml <<EOF
resources:
- ../../base
namePrefix: dev-
commonLabels:
  env: dev
images:
- name: myapp
  newTag: dev-latest
EOF

kustomize build myapp/overlays/dev/
```

---

## Exercise 3 — Prod overlay (more replicas, prod tag, namespace)

```bash
cat > myapp/overlays/prod/kustomization.yaml <<EOF
resources:
- ../../base
namePrefix: prod-
namespace: production
commonLabels:
  env: prod
replicas:
- name: myapp
  count: 6
images:
- name: myapp
  newName: myregistry.io/myapp
  newTag: v2.5.0
EOF

kustomize build myapp/overlays/prod/
```

> **Observe:** image becomes `myregistry.io/myapp:v2.5.0`, replicas becomes 6, name is `prod-myapp`, namespace is `production`.

---

## Exercise 4 — Use kustomize edit to set the image (CI/CD pattern)

```bash
cd myapp/overlays/prod
kustomize edit set image myapp=myregistry.io/myapp:v3.0.0
cat kustomization.yaml | grep -A3 images
kustomize build .
cd ../../..
```

---

## Exercise 5 — Apply both overlays to real namespaces

```bash
kubectl create namespace production
kubectl apply -k myapp/overlays/dev/ -n default
kubectl apply -k myapp/overlays/prod/

kubectl get deployments -A | grep myapp
```

---

## Exercise 6 — Verify commonLabels updated selectors correctly

```bash
kubectl get deployment prod-myapp -n production -o jsonpath='{.spec.selector}'
kubectl get deployment prod-myapp -n production -o jsonpath='{.metadata.labels}'
```

> **Observe:** both the selector AND the labels include `env: prod` — Kustomize kept them consistent (a manual sed/patch could easily have broken this).

---

## Exercise 7 — Cleanup

```bash
kubectl delete -k myapp/overlays/dev/ -n default
kubectl delete -k myapp/overlays/prod/
kubectl delete namespace production
rm -rf myapp
```

---

## Challenge Questions

1. Why does `commonLabels` update the Deployment's `selector` field, and why does that matter?
2. What is the practical CI/CD use of `kustomize edit set image`?
3. What happens if your overlay's `resources:` path is incorrect?
4. What's the difference between the `images` transformer and manually editing the image field in a patch?
5. Can a single base be used by more than one overlay simultaneously?

---

# Solutions

**Exercise 3 output (excerpt):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-myapp
  namespace: production
  labels:
    env: prod
spec:
  replicas: 6
  selector:
    matchLabels:
      app: myapp
      env: prod
  template:
    metadata:
      labels:
        app: myapp
        env: prod
    spec:
      containers:
      - image: myregistry.io/myapp:v2.5.0
        name: myapp
```

## Challenge Answers

**1. Why commonLabels updates selector?**
Kubernetes requires a Deployment's `spec.selector` to match its pod template's labels exactly — if commonLabels only updated metadata labels but not the selector/template labels, the Deployment would become invalid (selector wouldn't match its own pods). Kustomize's `commonLabels` transformer intentionally propagates to selectors and templates to keep the resource internally consistent.

**2. kustomize edit set image use case?**
CI/CD pipelines build a new image with a unique tag (often the git SHA or semantic version) after each commit. Rather than manually editing YAML, the pipeline runs `kustomize edit set image myapp=registry/myapp:$GIT_SHA` to programmatically update the overlay, then applies it — fully automatable without YAML string manipulation.

**3. Incorrect resources path?**
`kustomize build` fails immediately with an error like `rawResources failed to read Resources: ... no such file or directory` — it's a hard failure at render time, not a silent skip, so broken references are caught early before any cluster interaction.

**4. images transformer vs manual patch?**
The `images` transformer is purpose-built and simpler — it matches by image NAME (not the full string) across ALL containers/initContainers/resources automatically, regardless of how deeply nested. A manual JSON6902 patch would need an exact path to the image field in each specific resource, which is more brittle and verbose for something this common.

**5. One base, multiple overlays?**
Yes — this is the core design pattern. The base is read-only from the overlay's perspective; multiple overlays (dev, staging, prod) can all reference the same `../../base` simultaneously without conflict, each producing an independent rendered output.
