# 58 — Volume Mounting in Pods

## Overview

Every volume in Kubernetes follows the same pattern:
1. Declare the volume in `spec.volumes`
2. Mount it in a container via `spec.containers[].volumeMounts`

---

## Basic Mount Pattern

```yaml
spec:
  volumes:                         # ← Step 1: declare volumes
  - name: my-vol                   #   give it a name
    emptyDir: {}                   #   choose the type

  containers:
  - name: app
    image: nginx
    volumeMounts:                  # ← Step 2: mount into container
    - name: my-vol                 #   reference by volume name
      mountPath: /data             #   where in container filesystem
```

---

## volumeMount Fields

```yaml
volumeMounts:
- name: config-vol          # must match a volume name in spec.volumes
  mountPath: /etc/config    # absolute path inside the container
  subPath: app.conf         # optional: mount only one file from the volume
  readOnly: true            # optional: prevent writes (default: false)
  mountPropagation: None    # optional: how mounts propagate (None/HostToContainer/Bidirectional)
```

---

## subPath — Mount a Single File or Directory

Without `subPath`, the whole volume is mounted. With `subPath`, you can mount just one file:

```yaml
# Mount only app.conf from the ConfigMap
volumeMounts:
- name: config-vol
  mountPath: /etc/app/app.conf     # exact file path in container
  subPath: app.conf                # key name in ConfigMap

volumes:
- name: config-vol
  configMap:
    name: my-config
```

This is useful when:
- You want to inject a config file without replacing the whole directory
- Mounting a single Secret key as a file

---

## Multiple Volumes in One Pod

```yaml
spec:
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc
  - name: config
    configMap:
      name: app-config
  - name: creds
    secret:
      secretName: db-secret
  - name: tmp
    emptyDir: {}

  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: data
      mountPath: /var/data
    - name: config
      mountPath: /etc/config
      readOnly: true
    - name: creds
      mountPath: /etc/creds
      readOnly: true
    - name: tmp
      mountPath: /tmp/work
```

---

## Sharing Volumes Between Containers (Sidecar Pattern)

```yaml
spec:
  volumes:
  - name: shared-logs
    emptyDir: {}

  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app

  - name: log-shipper
    image: fluentd:latest
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app    # same path — reads what app writes
      readOnly: true
```

---

## Init Container + Volume Pattern

Init container pre-populates a volume before the app starts:

```yaml
spec:
  volumes:
  - name: init-data
    emptyDir: {}

  initContainers:
  - name: setup
    image: busybox
    command: ["sh", "-c", "echo 'config-value' > /init/config.txt"]
    volumeMounts:
    - name: init-data
      mountPath: /init

  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: init-data
      mountPath: /config       # reads what init container wrote
```

---

## readOnly Mounts

Always mount config and secret volumes as `readOnly: true` to prevent accidental writes:

```yaml
volumeMounts:
- name: creds
  mountPath: /etc/creds
  readOnly: true        # container cannot write to /etc/creds
```

---

## mountPropagation

Controls how mount events on the host propagate to containers:

| Mode | Behaviour |
|------|-----------|
| `None` (default) | No propagation — isolated |
| `HostToContainer` | New mounts on host appear in container |
| `Bidirectional` | Container mounts also appear on host (requires privileged) |

Most workloads use `None`.

---

## Useful Commands

```bash
# Check volume mounts in a running pod
kubectl describe pod mypod | grep -A20 "Mounts:"
kubectl get pod mypod -o jsonpath='{.spec.containers[0].volumeMounts}'

# See volumes declared on a pod
kubectl get pod mypod -o jsonpath='{.spec.volumes}'

# Exec into a pod and inspect mount
kubectl exec mypod -- df -h
kubectl exec mypod -- mount | grep /data
kubectl exec mypod -- ls -la /data
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| `name` in volumeMount doesn't match any volume | Pod fails with `MountVolume.SetUp failed` | Match names exactly |
| `mountPath` already exists in image | Content overwritten silently | Use `subPath` or different path |
| Missing `readOnly: true` on secret mounts | Security risk | Always set readOnly for secrets/configs |
| PVC not bound | Pod stuck `Pending` | Check PVC status and events |
