# Validating Admission Controllers — Practical Exercises

## Setup
```bash
kubectl create namespace vlab
kubectl config set-context --current --namespace=vlab
```

---

## Section A — Inspect Existing Webhooks

```bash
kubectl get validatingwebhookconfigurations
kubectl get validatingwebhookconfigurations -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.webhooks[*].clientConfig.service.name}{"\n"}{end}'
```

If anything is registered (cert-manager, kube-prometheus, Kyverno...):
```bash
kubectl get validatingwebhookconfiguration <name> -o yaml
```

---

## Section B — Write a CEL Policy

### B1. Require an `owner` label on every pod
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata: { name: require-owner-label }
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE","UPDATE"]
      resources: ["pods"]
  validations:
  - expression: "has(object.metadata.labels) && 'owner' in object.metadata.labels"
    message: "Pod must carry an 'owner' label."
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata: { name: require-owner-label-vlab }
spec:
  policyName: require-owner-label
  validationActions: ["Deny"]
  matchResources:
    namespaceSelector:
      matchLabels: { kubernetes.io/metadata.name: vlab }
EOF
```

### B2. Try to violate it
```bash
kubectl run no-owner --image=nginx
```
**Expected:** rejected with the message "Pod must carry an 'owner' label."

### B3. Comply
```bash
kubectl run with-owner --image=nginx --labels=owner=team-a
kubectl get pod with-owner
```
Created successfully.

### B4. Cleanup
```bash
kubectl delete pod with-owner
kubectl delete validatingadmissionpolicybinding require-owner-label-vlab
kubectl delete validatingadmissionpolicy require-owner-label
```

---

## Section C — Reject Specific Image Tags

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata: { name: no-latest-tags }
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE","UPDATE"]
      resources: ["pods"]
  validations:
  - expression: "object.spec.containers.all(c, !c.image.endsWith(':latest') && c.image.contains(':'))"
    message: "Container images must use a specific tag, not :latest."
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata: { name: no-latest-vlab }
spec:
  policyName: no-latest-tags
  validationActions: ["Deny"]
  matchResources:
    namespaceSelector:
      matchLabels: { kubernetes.io/metadata.name: vlab }
EOF

kubectl run latest-pod --image=nginx          # implicit :latest -> rejected
kubectl run pinned-pod --image=nginx:1.27     # accepted
kubectl delete pod pinned-pod
kubectl delete validatingadmissionpolicybinding no-latest-vlab
kubectl delete validatingadmissionpolicy no-latest-tags
```

---

## Section D — Test failurePolicy: Audit (warn instead of block)

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata: { name: warn-no-resources }
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE","UPDATE"]
      resources: ["pods"]
  validations:
  - expression: "object.spec.containers.all(c, has(c.resources.requests))"
    message: "Containers should declare resource requests."
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata: { name: warn-no-resources-vlab }
spec:
  policyName: warn-no-resources
  validationActions: ["Warn", "Audit"]      # NOT Deny — just inform
  matchResources:
    namespaceSelector:
      matchLabels: { kubernetes.io/metadata.name: vlab }
EOF

kubectl run nores --image=nginx
# Object is ACCEPTED, but kubectl prints a Warning.
kubectl delete pod nores
kubectl delete validatingadmissionpolicybinding warn-no-resources-vlab
kubectl delete validatingadmissionpolicy warn-no-resources
```

---

## Section E — Cleanup
```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace vlab
```
