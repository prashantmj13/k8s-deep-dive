# 75 — Kustomize Basics | Exercises

> **Format:** Conceptual/comparison exercises plus first look at kustomize
> **Estimated time:** 15 minutes

---

## Exercise 1 — Verify kustomize is available

```bash
kubectl version --client | grep -i kustomize
kustomize version 2>/dev/null || echo "kustomize not standalone-installed, kubectl has it built in"
```

---

## Exercise 2 — Compare a templated vs plain YAML file

Write out (don't apply yet) the same Deployment two ways:

**Helm-style (with template syntax):**
```yaml
replicas: {{ .Values.replicaCount }}
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

**Kustomize-style (always plain valid YAML):**
```yaml
replicas: 1
image: myapp:latest
```

Note that the Kustomize version is independently valid and renderable by `kubectl apply -f` directly — the Helm version is NOT valid YAML/Kubernetes syntax until rendered.

---

## Exercise 3 — Create a minimal base

```bash
mkdir -p myapp/base
cat > myapp/base/deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nginx:1.25
EOF

cat > myapp/base/kustomization.yaml <<EOF
resources:
- deployment.yaml
EOF

kubectl kustomize myapp/base/
```

---

## Exercise 4 — Decision exercise

For each scenario, decide Kustomize, Helm, or both, and justify:
1. You're publishing a database operator for others to install
2. Your team manages 20 internal microservices, each needing dev/staging/prod variants
3. You need to install the Prometheus stack from the community
4. You want GitOps with ArgoCD managing environment-specific overlays of your own apps

---

## Challenge Questions

1. Why is "template-free" a core design goal of Kustomize?
2. What's the practical risk of heavy Helm templating that Kustomize avoids?
3. Why can Kustomize bases be applied directly with `kubectl apply -f` even without an overlay?
4. Is `kubectl apply -k` the same as running standalone `kustomize build | kubectl apply -f -`?
5. Why do GitOps tools like ArgoCD/Flux favor Kustomize for environment overlays?

---

# Solutions

**Exercise 3 output:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - image: nginx:1.25
        name: myapp
```
(Identical to input since there's no overlay/patch yet — kustomize just re-serializes the resource.)

**Exercise 4:**
1. Helm — distributing to others requires packaging, versioning, and the broad ecosystem expectation that third-party software ships as a chart
2. Kustomize — you own all the YAML; overlays per environment are the textbook use case
3. Helm — Prometheus community charts are distributed via Helm, not Kustomize bases
4. Kustomize — ArgoCD/Flux have first-class native Kustomize support for exactly this pattern; many teams also use Helm+Kustomize together (helm template piped through kustomize)

## Challenge Answers

**1. Why template-free?**
Templating syntax embedded in YAML breaks YAML's own validity and readability — you can't lint, syntax-highlight, or directly apply a templated file without rendering it first. Kustomize keeps every file independently valid and inspectable, lowering cognitive overhead and tooling friction.

**2. Risk of heavy templating?**
Complex Helm templates with nested conditionals, loops, and `toYaml`/`nindent` gymnastics can become difficult to read and debug — the "what will actually get applied" question requires running `helm template` to find out. Errors often surface as obscure Go template errors rather than clear YAML/schema errors.

**3. Why bases work standalone?**
A Kustomize base is just a kustomization.yaml referencing ordinary, complete, valid Kubernetes manifests — there's no parameterization required for it to make sense. You CAN `kubectl apply -f` the raw files directly without ever invoking kustomize; the overlay/patch system is purely additive.

**4. apply -k vs kustomize build | apply?**
Functionally equivalent for most cases — `kubectl apply -k` invokes a built-in (often slightly older) version of kustomize logic. The standalone `kustomize` binary is updated more frequently and supports newer features first. For advanced features (e.g. newer transformers, `replacements`), use the standalone binary; for everyday use, `kubectl -k` is fine.

**5. Why GitOps tools favor Kustomize?**
GitOps relies on Git as the source of truth with manifests that are diffable and reviewable in pull requests. Kustomize's plain-YAML + patch model produces highly readable diffs (you see exactly which fields changed), whereas templated output requires rendering before you can meaningfully diff it — friction that GitOps workflows want to avoid.
