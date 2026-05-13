# 47 — Image Security | Solutions

---

## Exercise 1 — Secret type

```bash
kubectl get secret my-creds
```
```
NAME       TYPE                             DATA   AGE
my-creds   kubernetes.io/dockerconfigjson   1      5s
```

---

## Exercise 4 — Non-root pod

```bash
kubectl exec pod-secure -- id
# uid=1000 gid=0(root) groups=0(root)

kubectl exec pod-secure -- whoami
# 1000  (or: whoami: unknown uid 1000)
```

Pod runs as UID 1000, not root.

---

## Exercise 5 — Pull policy

```
nginx:latest  → Image Pull Policy: Always
nginx:1.25.3  → Image Pull Policy: IfNotPresent
```

---

## Challenge Answers

**1. Secret type for registry credentials?**
`kubernetes.io/dockerconfigjson`. It contains a base64-encoded JSON file in Docker config format: `{"auths":{"registry.example.com":{"auth":"base64(user:pass)"}}}`.

**2. imagePullPolicy values?**
- `Always`: Always pull the image, even if it exists locally. Good for `:latest` or mutable tags.
- `IfNotPresent`: Only pull if not already on the node. Default for tagged images. Saves bandwidth.
- `Never`: Never pull — fail if image not present locally. For airgapped environments.

**3. Never use :latest in production?**
`:latest` is a mutable tag — the same tag can point to different images over time. If a node already has a cached `:latest` and the image is updated in the registry, some nodes may run the old version and some the new. You lose reproducibility and auditability. Pin to specific digests (`sha256:abc...`) for true immutability.

**4. ImagePullSecret for all pods in namespace?**
Patch the `default` ServiceAccount (or any SA used by pods):
```bash
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "my-creds"}]}'
```
All pods using this SA automatically inherit the pull secret.

**5. runAsNonRoot: true with root image?**
The pod fails to start with: `Error: container has runAsNonRoot and image will run as root`. This is a safety check — if the Dockerfile USER is root (UID 0), Kubernetes blocks the container from starting. You must either fix the image to run as non-root or remove the constraint.
