# Validating Admission Controllers — Solutions

## Section B — Owner Label
```
$ kubectl run no-owner --image=nginx
Error from server: ValidatingAdmissionPolicy 'require-owner-label' with binding 'require-owner-label-vlab' denied request: Pod must carry an 'owner' label.
```
The denial includes the exact policy + binding name and the human-readable message you wrote. Easy to debug.

## Section C — No :latest
The CEL expression `object.spec.containers.all(c, !c.image.endsWith(':latest') && c.image.contains(':'))` checks every container. Pods using implicit `:latest` (e.g., `nginx`) are rejected; pinned ones (`nginx:1.27`) pass.

## Section D — Audit / Warn
Setting `validationActions: ["Warn","Audit"]` instead of `["Deny"]` lets policies be tested non-blockingly:
- `Warn` → kubectl shows a yellow warning to the user.
- `Audit` → emits an annotation in the audit log.
- `Deny` → reject the request.

This is invaluable for rolling out new policies without breaking existing workloads.

## Cheat Sheet

| Goal | Use |
|---|---|
| Built-in checks (privileged pod, quota) | Built-in admission plugins |
| Custom rules in CEL | `ValidatingAdmissionPolicy` (1.30+) |
| Custom rules in Rego | OPA Gatekeeper webhook |
| Custom rules in K8s-native YAML | Kyverno webhook |
| Custom logic too complex for CEL | Write a webhook server |

| Action | Effect |
|---|---|
| `Deny` | reject |
| `Warn` | accept but print a warning |
| `Audit` | accept but record in audit log |

| Field | Why it matters |
|---|---|
| `failurePolicy` | what happens if your validator is unavailable |
| `namespaceSelector` | restrict scope; avoid lockout |
| `timeoutSeconds` | webhook must respond fast (≤30s) |
| `sideEffects: None` | required for v1 |
