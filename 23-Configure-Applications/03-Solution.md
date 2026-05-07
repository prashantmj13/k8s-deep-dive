# Configure Applications — Solutions

## Section A — Integration Pod
The pod's logs show:

```
API_KEY=super-secret-key-12345          # from Secret via env
LOG_LEVEL=debug                          # literal env
greeting=Hello, world                    # ConfigMap key as env
app.conf=log_level: info\nlisten: 8080   # ConfigMap key as env (whole file as value)
KUBERNETES_*                             # auto-injected by K8s

----
log_level: info
listen: 8080                             # ConfigMap mounted as file
----
super-secret-key-12345                   # Secret mounted as file
```

**Key observations:**
- Both ConfigMap keys and the Secret's `api-key` appear as env vars (because of `envFrom` and `valueFrom`).
- Multi-line ConfigMap data (`app.conf`) becomes a multi-line env var. Awkward — usually you mount it as a file instead.
- Secret values are exposed in `env` as plain strings inside the container. Anyone with `exec` access into the pod can read them.

## Section B — Live ConfigMap Update
- Mounted **file** changes are propagated within ~60 seconds (the kubelet's sync interval).
- **Env vars** never update — they're snapshotted at pod creation. To pick up new values, you must restart the pod (e.g., `kubectl rollout restart`).

## Cheat Sheet

| Want | Use |
|---|---|
| Quick value, never changes | env literal |
| Value from non-secret data | env from ConfigMap |
| Value from sensitive data | env from Secret |
| Many vars from one source | envFrom |
| Settings file (nginx.conf, app.yaml) | ConfigMap as volume |
| Live updates without restart | Volume mount, app re-reads file |
| TLS keys, DB passwords | Secret as file (preferred over env) |
| Force restart after config change | `kubectl rollout restart deployment/X` |
