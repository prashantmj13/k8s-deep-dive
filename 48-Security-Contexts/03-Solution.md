# 48 — Security Contexts | Solutions

---

## Exercise 1 — User output

```bash
kubectl logs pod-user
# uid=1000 gid=3000 groups=2000,3000

kubectl exec pod-user -- id
# uid=1000 gid=3000 groups=2000,3000
```

---

## Exercise 3 — Read-only filesystem

```bash
kubectl exec pod-readonly -- touch /newfile
# touch: /newfile: Read-only file system   ← fails as expected

kubectl exec pod-readonly -- touch /tmp/newfile
# (succeeds — /tmp is an emptyDir volume)
```

---

## Exercise 5 — fsGroup

```bash
kubectl logs pod-fsgroup
# drwxrwsrwx  2 root 2000 /data   ← GID is 2000 (fsGroup)
```

The `s` in `drwxrwsrwx` is the setgid bit — new files in this directory automatically inherit GID 2000.

---

## Challenge Answers

**1. Pod-level vs container-level security context?**
Pod-level (`spec.securityContext`) applies to ALL containers in the pod as a default. Container-level (`spec.containers[].securityContext`) applies to that specific container and overrides the pod-level setting. Not all fields are available at both levels — `fsGroup` is pod-level only; `capabilities` and `readOnlyRootFilesystem` are container-level only.

**2. allowPrivilegeEscalation: false?**
Prevents the container process from gaining more privileges than its parent. Specifically it prevents: using `setuid` binaries (like sudo), privilege escalation via kernel bugs, and privilege escalation through `execve` with setuid/setgid. This is a critical hardening step.

**3. fsGroup?**
Sets the GID (group ID) that owns mounted volumes and is added to the container's supplemental groups. Kubernetes `chown`s volume contents to this GID at startup. This allows non-root containers to read/write volumes without needing root privileges.

**4. readOnlyRootFilesystem: true?**
If malware or an attacker gains code execution inside a container, a read-only filesystem prevents them from: downloading additional tools, modifying binaries, creating persistence scripts, or writing log/config files. It significantly limits what an attacker can do post-compromise.

**5. Safest capabilities approach?**
```yaml
capabilities:
  drop: ["ALL"]
  add: ["<only-what-app-needs>"]
```
Drop all capabilities first, then add back only the minimum required. Most applications need zero capabilities. Apps that bind to ports <1024 need `NET_BIND_SERVICE`. Network tools may need `NET_ADMIN`.
