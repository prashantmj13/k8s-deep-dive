# 76 — Kustomize Setup and Config | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 25 minutes

---

## Exercise 1 — Install standalone kustomize

```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/
kustomize version
```

---

## Exercise 2 — Build a basic kustomization

```bash
mkdir -p kust-lab && cd kust-lab
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
        image: nginx:1.25
EOF

cat > service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: demo
spec:
  selector: {app: demo}
  ports:
  - port: 80
EOF

cat > kustomization.yaml <<EOF
resources:
- deployment.yaml
- service.yaml
EOF

kustomize build .
```

---

## Exercise 3 — Add commonLabels and namePrefix

```bash
cat >> kustomization.yaml <<EOF
commonLabels:
  env: dev
  team: platform
namePrefix: dev-
EOF

kustomize build . | grep -E "name:|env:|team:"
```

---

## Exercise 4 — Add a configMapGenerator

```bash
cat >> kustomization.yaml <<EOF
configMapGenerator:
- name: demo-config
  literals:
  - LOG_LEVEL=debug
  - ENV=dev
EOF

kustomize build . | grep -A5 "kind: ConfigMap"
```

> **Observe:** the ConfigMap name has a hash suffix appended automatically.

---

## Exercise 5 — Set namespace for all resources

```bash
cat >> kustomization.yaml <<EOF
namespace: kust-lab-ns
EOF

kustomize build . | grep namespace
```

---

## Exercise 6 — Actually apply it

```bash
kubectl create namespace kust-lab-ns
kubectl apply -k .
kubectl get all -n kust-lab-ns
kubectl get configmap -n kust-lab-ns
```

---

## Exercise 7 — Diff before a change, then apply

```bash
sed -i 's/replicas: 1/replicas: 3/' deployment.yaml
kubectl diff -k .
kubectl apply -k .
kubectl get deployment -n kust-lab-ns
```

---

## Exercise 8 — Cleanup

```bash
kubectl delete -k .
kubectl delete namespace kust-lab-ns
cd .. && rm -rf kust-lab
```

---

## Challenge Questions

1. Why must the file be named exactly `kustomization.yaml`?
2. What does the hash suffix on a generated ConfigMap accomplish?
3. What's the difference between `kustomize build .` and `kubectl apply -k .`?
4. If `namespace:` is set in kustomization.yaml, does it override a namespace already specified inside a resource's metadata?
5. Why is `kubectl diff -k .` valuable before applying changes?

---

# Solutions

**Exercise 3:**
```
name: dev-demo
env: dev
team: platform
```

**Exercise 4:**
```
kind: ConfigMap
metadata:
  name: demo-config-a1b2c3d4
data:
  LOG_LEVEL: debug
  ENV: dev
```

**Exercise 6:** All resources (Deployment, Service, ConfigMap) appear in `kust-lab-ns`, prefixed `dev-`, labeled `env=dev,team=platform`.

## Challenge Answers

**1. Exact filename requirement?**
Kustomize specifically looks for a file named `kustomization.yaml` (or `.yml`/`Kustomization`) in the target directory — it's the single entry point that tells kustomize what to do. Without it, `kustomize build <dir>` fails immediately with "no kustomization file found."

**2. Hash suffix purpose?**
When a ConfigMap's content changes, its hash-based name changes too (e.g. `demo-config-a1b2c3d4` → `demo-config-e5f6g7h8`). Since Deployments reference the ConfigMap by name and Kustomize automatically updates that reference, changing the ConfigMap forces a new Deployment spec (new name referenced) — which triggers a rolling restart of pods to pick up the new config, solving the classic "ConfigMap changed but pods didn't restart" problem.

**3. build vs apply -k?**
`kustomize build .` only renders the final YAML to stdout — nothing touches the cluster. `kubectl apply -k .` renders AND applies the result to the cluster. Always run `build` first to review output before using `apply`.

**4. namespace override behavior?**
Yes — the `namespace:` field in kustomization.yaml forcibly sets/overrides the namespace on every resource it manages, even if a resource's YAML already specifies a different namespace in its `metadata.namespace`.

**5. Why kubectl diff -k matters?**
It shows exactly what would change in the live cluster (additions, removals, modifications) without applying anything — critical for catching unintended changes (e.g., an overlay accidentally removing a label that another controller depends on) before they affect a running environment, especially production.
