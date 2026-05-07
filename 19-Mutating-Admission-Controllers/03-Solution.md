# Mutating Admission Controllers — Solutions

## Section A — Existing Webhooks
On vanilla minikube/kind: empty list. On real clusters with cert-manager, Istio, Linkerd, Kyverno, vault-agent-injector, etc., you'll see entries. Each lists the rules (resources/operations it intercepts) and `clientConfig` (where to call).

## Section B — ServiceAccount Mutator
```yaml
spec:
  serviceAccountName: default              # injected by mutating admission
  serviceAccount: default                  # legacy alias, also injected
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: kube-api-access-xxxxx          # also injected
      mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      readOnly: true
  volumes:
  - name: kube-api-access-xxxxx            # also injected
    projected:
      sources:
      - serviceAccountToken: { path: token }
      - configMap: { name: kube-root-ca.crt, items: [{key: ca.crt, path: ca.crt}] }
      - downwardAPI: { items: [{path: namespace, fieldRef: {fieldPath: metadata.namespace}}] }
```
The user submitted a 5-line pod; the API persisted ~25 lines. **All of that came from a mutating admission controller** (the built-in `ServiceAccount` plugin).

## Section C — LimitRanger Mutator
After the LimitRange is in place, every new pod has its `resources` block populated:
```json
{
  "requests": {"cpu":"100m","memory":"128Mi"},
  "limits":   {"cpu":"200m","memory":"256Mi"}
}
```
This is also a mutator. The same controller does **both** mutation (defaulting) and validation (max/min checking).

## Section D — CEL Mutating
The CEL `Object{...}` expression returns a partial object that is merged into the existing one. The example adds `metadata.labels.team: payments` to every new pod. This is the future of simple mutations — no webhook server needed.

## Section E — Istio Injection
Istio's mutating webhook adds:
- An `istio-init` initContainer (sets up iptables for traffic interception).
- An `istio-proxy` sidecar (Envoy).
- Volume mounts for certs and config.
- Labels (`security.istio.io/tlsMode`, `service.istio.io/canonical-name`, etc.).

The user-submitted pod had 1 container; the persisted pod has 2 containers + 1 init container.

---

## Cheat Sheet

| Action | What it does |
|---|---|
| Inject sidecars | Mutating webhook (Istio, Linkerd) |
| Default fields | Built-in mutator (LimitRanger, ServiceAccount, DefaultStorageClass) |
| Override image | Mutating webhook (rewrite `image:` to internal registry) |
| Tag with labels | Mutating webhook or CEL `MutatingAdmissionPolicy` |

| Field | Why it matters |
|---|---|
| `failurePolicy: Ignore` | safer for non-critical mutators |
| `reinvocationPolicy: IfNeeded` | re-runs after other mutators changed the object |
| `namespaceSelector` | restrict scope; avoid cluster lockout |
| Idempotent logic | mutators may run multiple times — design for it |

| Order recap | Mutating → Schema validation → Validating → etcd |
|---|---|

---

All 16 folders done. The full repo structure is:

```
Kubernetes-Architecture/
Containers-and-Pods/
ReplicaSet/
Deployments/
Manual-Pod-Scheduling/
Labels-and-Selectors/
Taints-and-Tolerations/
Node-Selector/
Node-Affinity/
Resource-Requirements/
Resource-Limits/
DaemonSets/
Static-Pods/
Priority-Classes/
Multiple-Schedulers/
Scheduler-Profile/
Admission-Controllers/
Validating-Admission-Controllers/
Mutating-Admission-Controllers/
```

Each folder has `01-Theory.md`, `02-Exercise.md`, `03-Solution.md`, plus an `images/` directory with SVG diagrams.
