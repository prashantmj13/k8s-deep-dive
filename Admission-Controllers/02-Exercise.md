# Admission Controllers — Practical Exercises

## Setup
```bash
kubectl create namespace aclab
kubectl config set-context --current --namespace=aclab
```

---

## Section A — Inspect Active Built-ins

For minikube:
```bash
minikube ssh
sudo grep enable-admission /etc/kubernetes/manifests/kube-apiserver.yaml
exit
```
Look for `--enable-admission-plugins=NodeRestriction,...` (the default in 1.25+).

---

## Section B — See LimitRanger in Action (Mutating)

### B1. Apply a LimitRange
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata: { name: defaults }
spec:
  limits:
  - type: Container
    defaultRequest: { cpu: "100m", memory: "128Mi" }
    default:        { cpu: "500m", memory: "512Mi" }
EOF
```

### B2. Create a pod with NO resources
```bash
kubectl run no-res --image=nginx
kubectl get pod no-res -o jsonpath='{.spec.containers[0].resources}' | python3 -m json.tool
```
**Expected:** `requests` and `limits` populated by the admission controller, even though we didn't specify them.

### B3. Cleanup
```bash
kubectl delete pod no-res
kubectl delete limitrange defaults
```

---

## Section C — See ResourceQuota in Action (Validating)

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata: { name: ns-cap }
spec:
  hard:
    requests.cpu: "500m"
    requests.memory: "512Mi"
    pods: "5"
EOF

# Try to over-budget
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: too-fat }
spec:
  containers:
  - name: c
    image: nginx
    resources:
      requests: { cpu: "1", memory: "100Mi" }
EOF
```
**Expected:** rejected by ResourceQuota admission with `exceeded quota`.

Cleanup:
```bash
kubectl delete pod too-fat --ignore-not-found
kubectl delete resourcequota ns-cap
```

---

## Section D — See PodSecurity in Action

### D1. Make the namespace "restricted"
```bash
kubectl label namespace aclab pod-security.kubernetes.io/enforce=restricted --overwrite
```

### D2. Try to create a privileged pod
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: priv }
spec:
  containers:
  - name: c
    image: nginx
    securityContext:
      privileged: true
EOF
```
**Expected:** rejected by `PodSecurity` admission with violations listed.

### D3. Reset
```bash
kubectl label namespace aclab pod-security.kubernetes.io/enforce-
```

---

## Section E — Inspect Webhooks

```bash
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations
```
On a default minikube/kind these may be empty. On clusters with cert-manager, Istio, OPA, etc., you'll see entries.

```bash
# If anything is registered, inspect:
kubectl get mutatingwebhookconfiguration <name> -o yaml
```
Notable fields: `clientConfig.service`, `rules`, `failurePolicy`, `namespaceSelector`, `caBundle`.

---

## Section F — Try a CEL ValidatingAdmissionPolicy

(Requires Kubernetes 1.30+ for stable; 1.26+ for beta.)
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata: { name: deny-large-mem }
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["pods"]
  validations:
  - expression: |
      !object.spec.containers.exists(c, has(c.resources.requests.memory) && quantity(c.resources.requests.memory).isGreaterThan(quantity('100Mi')))
    message: "Memory request must be <= 100Mi in this lab."
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata: { name: deny-large-mem-binding }
spec:
  policyName: deny-large-mem
  validationActions: ["Deny"]
  matchResources:
    namespaceSelector:
      matchLabels: { kubernetes.io/metadata.name: aclab }
EOF

# Now try a too-big pod
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: huge-mem }
spec:
  containers:
  - name: c
    image: nginx
    resources: { requests: { memory: "200Mi" } }
EOF
```
**Expected:** rejected with `Memory request must be <= 100Mi in this lab.`

Cleanup:
```bash
kubectl delete pod huge-mem --ignore-not-found
kubectl delete validatingadmissionpolicybinding deny-large-mem-binding
kubectl delete validatingadmissionpolicy deny-large-mem
```

---

## Section G — Cleanup
```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace aclab
```
