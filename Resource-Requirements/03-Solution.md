# Resource Requests — Solutions

---

## Section A — Requests Set
The pod's resources block is preserved exactly as you submitted it. QoS class is `Burstable` because requests are set but no limits exist.

## Section B — Doesn't Fit
```
0/1 nodes are available: 1 Insufficient memory.
```
Even a 16-Gi node cannot satisfy 500-Gi requests. The scheduler refuses to bind the pod.

## Section C — Allocatable
```
Allocated resources:
  Resource           Requests   Limits
  cpu                400m       1100m
  memory             528Mi      812Mi
  ephemeral-storage  0          0
```
The kubelet's view of capacity vs. what's been promised.

## Section D — QoS
- `guar`: Guaranteed (every container has both, equal)
- `burst`: Burstable
- `effort`: BestEffort

## Section E — Cgroups
A pod with `cpu: 200m` gets `cpu.weight: ~20` (cgroup v2 maps weights 1-10000 from a base of 100 per CPU unit). Pods with `cpu: 1` get a much higher weight. During contention, the kernel divides CPU in proportion to weight.

## Section F — LimitRange
LimitRange adds defaults at admission time. After it applies:
```json
"resources": {
  "requests": {"cpu":"100m","memory":"128Mi"},
  "limits":   {"cpu":"500m","memory":"512Mi"}
}
```
This is how cluster operators ensure no pod arrives without explicit resources.

## Section G — ResourceQuota
The API server rejects with:
```
forbidden: exceeded quota: ns-cap, requested: requests.cpu=2, used: requests.cpu=0, limited: requests.cpu=1
```
Quota is enforced at admission. Existing pods are unaffected when you create a new quota object that they would have violated; new admissions must comply.

---

## Cheat Sheet
| Field | Value example | Meaning |
|---|---|---|
| `requests.cpu` | `250m`, `0.5`, `1` | guaranteed CPU floor |
| `requests.memory` | `128Mi`, `2Gi`, `1G` | memory floor (not enforced at runtime) |
| `requests.ephemeral-storage` | `1Gi` | local disk request |
| QoS | derived | Guaranteed / Burstable / BestEffort |
