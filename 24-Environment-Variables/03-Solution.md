# Environment Variables — Solutions

## Section A — Literals
```
GREETING=Hello World
LOG_LEVEL=info
PORT=8080
```
Plus all the K8s-injected ones (KUBERNETES_*, HOSTNAME, etc.).

## Section B — ConfigMap
```
APP_cache=redis
APP_database=postgres
DB=postgres
```
The `valueFrom` produced `DB=postgres`. The `envFrom` with `prefix: APP_` produced `APP_database` and `APP_cache`.

## Section C — Secret
```
PASS=s3cret
USER=admin
```
The values are in plaintext inside the container. Anyone with `kubectl exec` can read them. This is why **mounted Secret volumes are preferred** for security-sensitive workloads.

## Section D — Downward API
```
NODE_NAME=minikube
POD_IP=10.244.0.5
POD_NAME=downward
POD_NAMESPACE=envlab
```
The pod knows its own identity without API calls. Useful for logging, distributed tracing, leader election.

## Section E — Resource Field
```
CPU_REQUEST=200
MEM_LIMIT=512
```
With `divisor: 1m`, CPU is in millicores. With `divisor: 1Mi`, memory is in MiB. The JVM, Node, Go runtimes can use these to size threadpools/heaps.

## Section F — Stale
```
greeting=hello       # before edit
greeting=hello       # AFTER editing the ConfigMap — still hello!
greeting=hello
```
The env var is set once at container start. Editing the ConfigMap does not update env vars. Only restarting the pod does.

For live updates, mount the ConfigMap as a volume — files are eventually consistent, app re-reads.

---

## Cheat Sheet

| Source | Snippet |
|---|---|
| Literal | `value: "info"` |
| ConfigMap key | `valueFrom: { configMapKeyRef: { name: X, key: Y } }` |
| Secret key | `valueFrom: { secretKeyRef: { name: X, key: Y } }` |
| Pod field | `valueFrom: { fieldRef: { fieldPath: metadata.name } }` |
| Resource | `valueFrom: { resourceFieldRef: { resource: limits.memory, divisor: 1Mi } }` |
| Bulk | `envFrom: [{ configMapRef: { name: X } }, { secretRef: { name: Y } }]` |
| Optional | add `optional: true` so missing CM/Secret doesn't block startup |

| Property | env vars |
|---|---|
| Update on source change | NO |
| Visible inside container | yes (`printenv`) |
| Visible in `kubectl describe` | yes (literals) / no (valueFrom names only) |
| Best for secrets | NO — prefer file mounts |
