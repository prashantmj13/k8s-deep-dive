# 48 — Security Contexts

## What are Security Contexts?

A Security Context defines **privilege and access control settings** for a Pod or Container. It controls what the container process can and cannot do on the host.

Security contexts can be set at two levels:
- **Pod level** (`spec.securityContext`) — applies to all containers
- **Container level** (`spec.containers[].securityContext`) — overrides pod level for that container

---

## Key Security Context Fields

### Pod-level (`spec.securityContext`)

```yaml
securityContext:
  runAsUser: 1000           # UID to run all containers as
  runAsGroup: 3000          # GID for all containers
  fsGroup: 2000             # GID for volume ownership
  runAsNonRoot: true        # fail if image runs as root
  supplementalGroups: [4000]  # extra GIDs
  sysctls:                  # kernel parameters
  - name: net.core.somaxconn
    value: "1024"
```

### Container-level (`spec.containers[].securityContext`)

```yaml
securityContext:
  runAsUser: 1000
  runAsNonRoot: true
  allowPrivilegeEscalation: false  # prevent sudo/setuid
  readOnlyRootFilesystem: true     # immutable container filesystem
  privileged: false                # never true in production
  capabilities:
    add: ["NET_ADMIN", "SYS_TIME"]
    drop: ["ALL"]                  # drop all, add only what's needed
```

---

## Linux Capabilities

Capabilities break up root privileges into granular units:

| Capability | Allows |
|-----------|--------|
| `NET_ADMIN` | Network interface configuration |
| `SYS_TIME` | Set system clock |
| `SYS_PTRACE` | Process tracing (debug) |
| `NET_BIND_SERVICE` | Bind to ports < 1024 |
| `CHOWN` | Change file ownership |

Best practice: **drop ALL, add only what's needed**:

```yaml
securityContext:
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]
```

---

## Examples

### Run as specific user

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "id && sleep 3600"]
```

### Read-only filesystem with writable tmp

```yaml
containers:
- name: app
  image: nginx
  securityContext:
    readOnlyRootFilesystem: true
  volumeMounts:
  - name: tmp
    mountPath: /tmp
  - name: var-run
    mountPath: /var/run
volumes:
- name: tmp
  emptyDir: {}
- name: var-run
  emptyDir: {}
```

### Privileged container (avoid in production)

```yaml
securityContext:
  privileged: true   # gives full host access — dangerous
```

---

## fsGroup — Volume Ownership

`fsGroup` sets the GID for any mounted volumes. Files created in the volume are owned by this GID:

```yaml
spec:
  securityContext:
    fsGroup: 2000
  containers:
  - name: app
    volumeMounts:
    - name: data
      mountPath: /data
```

```bash
kubectl exec mypod -- ls -la /data
# drwxrwsr-x  2 root 2000  /data   ← owned by GID 2000
```

---

## Pod Security Admission (PSA) — Cluster-wide Enforcement

PSA enforces security standards at namespace level:

```bash
# Label a namespace to enforce restricted policy
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted
```

| Level | What it enforces |
|-------|-----------------|
| `privileged` | No restrictions |
| `baseline` | Prevents known privilege escalations |
| `restricted` | Strict: non-root, no capabilities, read-only FS recommended |

---

## Useful Commands

```bash
# Check what user a container is running as
kubectl exec mypod -- id
kubectl exec mypod -- whoami

# Check capabilities
kubectl exec mypod -- cat /proc/1/status | grep Cap
```
