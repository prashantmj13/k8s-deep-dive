# Labels and Selectors — Solutions

---

## Section B — Equality

| Selector | Pods returned |
|---|---|
| `app=web` | p1, p2, p3, p4 |
| `app=web,env=prod` | p1, p2, p4 |
| `env!=prod` | p3, p6, p8 |
| `app=web,env=prod,tier=frontend` | p1, p2 |

Equality selectors are AND-ed when you list multiple, separated by commas.

## Section C — Set-Based

| Selector | Pods returned |
|---|---|
| `'app in (web,api)'` | p1..p6 |
| `'tier notin (data)'` | p1..p6 |
| `'owner'` (after labeling p1) | only p1 |
| `'!owner'` | every pod except p1 |

`Exists` and `DoesNotExist` are powerful — they let you tag a subset of pods with a marker key, then operate on "tagged" or "untagged" pods.

## Section D — Bulk

`kubectl label pods -l app=web team=alpha` adds `team=alpha` to all 4 web pods in one shot. This pattern is powerful for one-time migrations and tagging.

`kubectl delete pods -l env=staging` deletes 3 pods in one command.

## Section E — Service Selector

The Service's selector is evaluated continuously. The kube-controller-manager's **EndpointSlice controller** watches pods, finds those matching the Service's selector, and writes their IPs into an EndpointSlice object. kube-proxy then uses that to program iptables/IPVS.

Empty selector → empty EndpointSlice → traffic is dropped (or returns connection refused). Always `kubectl get endpoints` to debug.

## Section F — Label Mutation

- `kubectl label pod X k=v` adds; errors if `k` already exists.
- `kubectl label pod X k=v --overwrite` updates.
- `kubectl label pod X k-` removes (trailing dash).
- All three accept `-l selector` to apply across many objects.

## Section G — Annotations

Annotations have **no index** in etcd. They cannot be used for selection. They are for tools that read by name and use them as metadata (e.g., `prometheus.io/scrape: "true"` is read by Prometheus when it inspects each pod).

## Section H — Canonical Labels

These are not magic — they are conventions the community settled on. Helm, Kustomize, ArgoCD, and many dashboards rely on them. If you label your apps this way, third-party tooling "just works."

---

## Cheat Sheet

| Operator | Equality form | Set form |
|---|---|---|
| Equal | `key=value` | `key in (v)` |
| Not equal | `key!=value` | `key notin (v)` |
| Has key | n/a | `key` |
| Lacks key | n/a | `!key` |

| Goal | Command |
|---|---|
| Add label | `kubectl label X k=v` |
| Update | `kubectl label X k=v --overwrite` |
| Remove | `kubectl label X k-` |
| Show | `kubectl get X --show-labels` or `-L k1,k2` |
| Filter | `kubectl get X -l 'app in (a,b),env=prod'` |
| Bulk apply | `kubectl <action> X -l selector` |
