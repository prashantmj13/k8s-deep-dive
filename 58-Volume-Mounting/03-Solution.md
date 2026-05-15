# 58 — Volume Mounting | Solutions

---

## Exercise 2 — Multi-volume output

```bash
kubectl logs pod-multi-vol
```
```
=== Config ===
env
log_level
=== Secret ===
api_key
=== Temp ===

```

Each ConfigMap key (`env`, `log_level`) and each Secret key (`api_key`) becomes a separate file. The `emptyDir` at `/tmp/work` is empty initially.

---

## Exercise 3 — subPath result

```bash
kubectl logs pod-subpath
# server { listen 80; }

kubectl exec pod-subpath -- ls /etc/nginx
# nginx.conf
```

Only `nginx.conf` exists at that path. Without `subPath`, the entire ConfigMap would replace the directory content and all other files nginx expects (like `mime.types`, `conf.d/`) would be gone.

---

## Exercise 4 — Sidecar logs

```bash
kubectl logs pod-sidecar -c reader
```
```
Tue May 13 10:00:03 UTC 2026
Tue May 13 10:00:06 UTC 2026
Tue May 13 10:00:09 UTC 2026
```

Reader sees timestamps as they're written by the writer — shared emptyDir.

---

## Exercise 5 — readOnly enforcement

```bash
kubectl exec pod-multi-vol -- touch /etc/secret/newfile
# touch: /etc/secret/newfile: Read-only file system

kubectl exec pod-multi-vol -- touch /tmp/work/newfile
# (succeeds — emptyDir is writable)
kubectl exec pod-multi-vol -- ls /tmp/work
# newfile
```

---

## Exercise 6 — Mount inspection

```bash
kubectl exec pod-multi-vol -- mount | grep -E "/etc/config|/etc/secret"
```
```
tmpfs on /etc/config type tmpfs (ro,relatime,...)   ← ro = read-only
tmpfs on /etc/secret type tmpfs (ro,relatime,...)   ← ro = read-only
tmpfs on /tmp/work type tmpfs (rw,relatime,...)     ← rw = read-write
```

---

## Challenge Answers

**1. subPath and when to use it?**
`subPath` mounts a single file or directory from the volume instead of the entire volume. Use it when: you want to inject one config file into an existing directory without overwriting other files, or when mounting multiple ConfigMap/Secret keys into different files at specific paths.

**2. readOnly: true on secret mounts?**
If a container is compromised, `readOnly: true` prevents the attacker from overwriting credentials or injecting malicious content. It also prevents accidental writes from application bugs. Secrets and ConfigMaps should always be mounted read-only — your application should only read, never write, to these mounts.

**3. Two containers, same volume, different paths?**
Yes. Both containers reference the same volume name, but each can use a different `mountPath`. They see the same data, just at different paths inside their respective containers. This is the sidecar pattern.

**4. mountPath already has files from the image?**
The volume **replaces** (shadows) whatever the image has at that path. The original image files at `mountPath` become inaccessible. This is why `subPath` is important for config injection — without it, you risk replacing a whole directory with just your config files and breaking the application.

**5. Verify mounts inside running container?**
```bash
kubectl exec mypod -- df -h          # shows all mounted filesystems
kubectl exec mypod -- mount          # detailed mount table
kubectl exec mypod -- ls /data       # verify contents
kubectl describe pod mypod | grep -A20 "Mounts:"   # from kubectl
```
