# 57 — Volume Types | Solutions

---

## Exercise 2 — emptyDir inter-container

```bash
kubectl logs pod-emptydir -c reader
```
```
Tue May 13 10:00:00 UTC 2026
Tue May 13 10:00:02 UTC 2026
Tue May 13 10:00:04 UTC 2026
```

The reader container sees timestamps written by the writer container — they share the same emptyDir.

---

## Exercise 3 — hostPath

```bash
kubectl exec pod-hostpath -- ls /host-tmp | head -10
```
```
ansible_tmp_xxx
k8s-tmp-yyy
snap.xxx
systemd-xxx
```

These are actual files from the node's `/tmp` directory — the container is reading the host filesystem.

---

## Exercise 4 — ConfigMap volume

```bash
kubectl logs pod-configmap-vol
```
```
database_host
log_level
db.example.com
```

Lines 1-2: `ls /etc/config` — each ConfigMap key is a separate file.
Line 3: `cat /etc/config/database_host` — file content is the value.

---

## Exercise 5 — Secret volume

```bash
kubectl logs pod-secret-vol
```
```
-r--------  1 root root 7 /etc/creds/password
S3cr3t!
```

File permissions are `0400` (owner read-only). Secret value is decoded automatically by kubelet.

---

## Exercise 6 — Memory-backed emptyDir

```bash
kubectl logs pod-mem-emptydir
```
```
Filesystem      Size  Used Avail Use% Mounted on
tmpfs            64M     0   64M   0% /ramdisk
```

`tmpfs` confirms it's RAM-backed. Faster than disk but counts against the container's memory limit.

---

## Challenge Answers

**1. emptyDir vs hostPath?**
- `emptyDir`: Created fresh for each pod, shared between containers in that pod, deleted when pod terminates. Data never touches the host filesystem in a predictable location.
- `hostPath`: Mounts an existing directory from the node's filesystem. Data persists after pod termination (it's on the host). Different nodes may have different content at the same path.

**2. emptyDir medium: Memory use cases?**
When you need very fast scratch storage: inter-process communication via shared memory, ML model inference caching, high-throughput temporary file processing, or any workload where disk I/O is the bottleneck. The tradeoff is it counts toward the container's memory limit and is lost on node restart.

**3. hostPath security risk?**
A container with hostPath access can read or write sensitive host files: `/etc/passwd`, `/var/run/docker.sock` (container escape), kernel modules, SSH keys, etc. If the container is compromised, the attacker can pivot to the host. Use only for DaemonSets that legitimately need host access (log collectors, monitoring agents, CSI drivers).

**4. ConfigMap volume vs env vars?**
- Volume mount: Each key becomes a file. Files **auto-update** when ConfigMap changes (within ~60s). Can mount large configs, certificates, multi-line files. App must read from filesystem.
- Env vars: Values injected at pod startup. **Do NOT update** — pod must restart to see changes. Simple key-value. App reads from environment.

**5. hostPath vs local PV type?**
- `hostPath`: Simple, no scheduling awareness. A pod using hostPath can land on any node and see different data on each node.
- `local` PV: Has `nodeAffinity` — Kubernetes guarantees the pod is scheduled to the specific node where the disk resides. Used with `WaitForFirstConsumer` StorageClass. Proper PVC/PV lifecycle management. Safer and correct for local NVMe/SSD usage.
