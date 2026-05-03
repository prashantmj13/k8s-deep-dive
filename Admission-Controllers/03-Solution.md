# Admission Controllers — Solutions

## Section A — Active Built-ins
Modern kubeadm clusters typically enable: `NamespaceLifecycle, LimitRanger, ServiceAccount, NodeRestriction, ResourceQuota, DefaultStorageClass, MutatingAdmissionWebhook, ValidatingAdmissionWebhook, PodSecurity` and a few more.

## Section B — LimitRanger Mutates
After the LimitRange exists, every new pod without explicit resources gets:
```json
{
  "limits":   {"cpu":"500m","memory":"512Mi"},
  "requests": {"cpu":"100m","memory":"128Mi"}
}
```
This is **mutation** — the API persisted a different object than the user submitted.

## Section C — ResourceQuota Rejects
```
forbidden: exceeded quota: ns-cap, requested: requests.cpu=1, used: requests.cpu=0, limited: requests.cpu=500m
```
This is **validation** — the object is unchanged but rejected.

## Section D — PodSecurity Rejects
```
violates PodSecurity "restricted:latest": privileged (container "c" must not set securityContext.privileged=true)
```
PodSecurity is the modern, built-in replacement for the deprecated PodSecurityPolicy.

## Section E — Webhooks
On vanilla minikube/kind: empty. On real clusters: many. Each entry tells you the webhook URL, the resources it watches, and its `failurePolicy`.

## Section F — CEL Policy
ValidatingAdmissionPolicy (1.30 GA) lets you write CEL rules without running a webhook server. The policy + binding model lets you enable rules per-namespace, per-resource. This is much cheaper than webhooks.

## Cheat Sheet

| Type | Modifies object? | Examples |
|---|---|---|
| Mutating built-in | yes | LimitRanger, ServiceAccount, DefaultStorageClass |
| Validating built-in | no | ResourceQuota, PodSecurity, NamespaceLifecycle |
| MutatingAdmissionWebhook | yes | Istio sidecar injection |
| ValidatingAdmissionWebhook | no | OPA Gatekeeper, Kyverno |
| ValidatingAdmissionPolicy (CEL) | no | Modern lightweight policies |

| Order | Mutating -> Schema validation -> Validating -> etcd |
|---|---|

| Pipeline | Authn -> Authz -> Mutating -> Validating -> Persist |
