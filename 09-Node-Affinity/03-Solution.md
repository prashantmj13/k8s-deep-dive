# Node Affinity — Solutions

---

## Section A — Required
- `zoned`: runs on the labeled node (passes "zone in [a,b]").
- `ghost-zone`: stays Pending; events show `0/N nodes are available: N node(s) didn't match Pod's node affinity/selector.`

## Section B — Operators
- **Exists** matches any node that has the key, regardless of value.
- **NotIn** is satisfied even if the key is missing entirely (the missing key trivially is not in the value list).
- **DoesNotExist** matches nodes that LACK the key. Useful for excluding "deprecated" or "tainted-by-label" nodes.

## Section C — Preferred Fallback
**Lesson:** soft rules never block. Even when no node has the `gpu` label, the pod still schedules — just with a lower placement score. Compare with `required: Exists` which would have left the pod Pending.

## Section D — Combo
The `required` is the gate. The two `preferred` rules sum to a score per node:
- A node with `disktype=ssd` AND `zone=us-east-1a` scores 100.
- A node with only `disktype=ssd` scores 80.
- A node with neither scores 0 but is still feasible (because `required` is met).

## Section E — OR vs AND
The pod runs if (ssd + zone-a) **OR** (zone-b). Term-level OR; expression-level AND. This is the only way to express OR in `nodeAffinity` — duplicate the whole term.

---

## Cheat Sheet

| Construct | Meaning |
|---|---|
| `requiredDuringSchedulingIgnoredDuringExecution` | Hard constraint; Pending if unmet |
| `preferredDuringSchedulingIgnoredDuringExecution` | Soft scoring; weight 1-100 |
| `nodeSelectorTerms` (list) | OR between terms |
| `matchExpressions` (list inside term) | AND inside a term |
| operators | `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt` |
| IgnoredDuringExecution | Once placed, label changes don't move the pod |
