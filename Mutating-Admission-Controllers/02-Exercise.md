# Mutating Admission Controllers — Practical Exercises

## Setup
```bash
kubectl create namespace mlab
kubectl config set-context --current --namespace=mlab
```

---

## Section A — Inspect Existing Mutating Webhooks

```bash
kubectl get mutatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
```

If anything is registered (cert-manager, Istio, Kyverno, etc.):
```bash
kubectl get mutatingwebhookconfiguration <name> -o yaml
```

---

## Section B — Watch the Built-in `ServiceAccount` Mutator

```bash
kubectl run plain --image=nginx
kubectl get pod plain -o yaml | grep -A2 serviceAccount
kubectl get pod plain -o yaml | grep -B1 -A3 'token'
```
**Expected:** `serviceAccountName: default` was added (you didn't set it). A projected token volume was also injected. This is the built-in `ServiceAccount` mutating admission controller in action.

Cleanup:
```bash
kubectl delete pod plain
```

---

## Section C — Watch `LimitRanger` Inject Defaults

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata: { name: defaults }
spec:
  limits:
  - type: Container
    defaultRequest: { cpu: "100m", memory: "128Mi" }
    default:        { cpu: "200m", memory: "256Mi" }
EOF

kubectl run no-res --image=nginx
kubectl get pod no-res -o jsonpath='{.spec.containers[0].resources}' | python3 -m json.tool
```
**Expected:** the resources block was injected.

Cleanup:
```bash
kubectl delete pod no-res
kubectl delete limitrange defaults
```

---

## Section D — Add Default Labels via CEL (1.32+)

If your cluster is 1.32+ with `MutatingAdmissionPolicy` alpha enabled (feature gate `MutatingAdmissionPolicy=true`):
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: MutatingAdmissionPolicy
metadata: { name: add-team-label }
spec:
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE"]
      resources: ["pods"]
  failurePolicy: Fail
  mutations:
  - patchType: ApplyConfiguration
    applyConfiguration:
      expression: |
        Object{
          metadata: Object.metadata{
            labels: { "team": "payments" }
          }
        }
EOF
```
On older clusters, this is unavailable — use a webhook instead.

---

## Section E — Inspect a Real Sidecar Injection

If you have Istio installed:
```bash
kubectl label namespace mlab istio-injection=enabled --overwrite
kubectl run app --image=nginx
kubectl get pod app -o jsonpath='{.spec.containers[*].name}'
# expects: app istio-proxy
kubectl get pod app -o jsonpath='{.spec.initContainers[*].name}'
# expects: istio-init
```
This is mutating admission injecting two extra containers and a label.

Cleanup:
```bash
kubectl delete pod app
kubectl label namespace mlab istio-injection-
```

---

## Section F — Cleanup
```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace mlab
```
