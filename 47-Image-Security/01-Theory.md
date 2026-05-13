# 47 — Image Security

## Why Image Security Matters

Container images are the primary attack surface. Compromised or malicious images can:
- Contain vulnerable libraries (CVEs)
- Run as root inside containers
- Exfiltrate data or mine cryptocurrency
- Gain access to the host via container escapes

---

## Private Registry Authentication

By default Kubernetes can only pull from public registries. For private registries, you need an `imagePullSecret`.

### Create a docker-registry Secret

```bash
kubectl create secret docker-registry my-registry-creds \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myuser@example.com
```

### Use it in a Pod

```yaml
spec:
  imagePullSecrets:
  - name: my-registry-creds
  containers:
  - name: app
    image: registry.example.com/myapp:v1.2.3
```

### Attach to a Service Account (all pods using SA get pull access)

```bash
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "my-registry-creds"}]}'
```

---

## Image Naming Best Practices

```
registry.example.com/team/app:v1.2.3
└──────────────────┘ └───────┘ └─────┘
      registry          repo     tag

# Never use :latest in production
# Always pin to a specific digest for immutability:
image: registry.example.com/myapp@sha256:abc123...
```

---

## Security Best Practices for Images

### 1. Use minimal base images
```dockerfile
# Instead of ubuntu:latest (200MB+)
FROM gcr.io/distroless/static:nonroot   # ~2MB, no shell
FROM alpine:3.19                         # ~5MB
```

### 2. Run as non-root
```dockerfile
FROM alpine:3.19
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

```yaml
# Enforce in pod spec
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
```

### 3. Use read-only root filesystem
```yaml
securityContext:
  readOnlyRootFilesystem: true
```

### 4. Never store secrets in images
Use Secrets or external secret managers (Vault, AWS Secrets Manager) instead.

### 5. Scan images for vulnerabilities
```bash
# Trivy (open source)
trivy image nginx:latest

# In CI/CD pipeline
trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:latest
```

---

## Admission Controllers for Image Security

### ImagePolicyWebhook

Enforces image policies (allowed registries, required tags, scan results) via a webhook:

```bash
# Enable on API server
--enable-admission-plugins=...,ImagePolicyWebhook
--admission-control-config-file=/etc/kubernetes/admission-config.yaml
```

### OPA/Kyverno Policies

Example Kyverno policy — only allow images from approved registry:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  rules:
  - name: validate-registries
    match:
      resources:
        kinds: [Pod]
    validate:
      message: "Only images from registry.example.com are allowed"
      pattern:
        spec:
          containers:
          - image: "registry.example.com/*"
```

---

## Checking Image Pull Status

```bash
kubectl describe pod mypod | grep -E "Image:|Pull|Failed"

# Events section shows:
# Normal   Pulling    kubelet  Pulling image "registry.example.com/app:v1"
# Normal   Pulled     kubelet  Successfully pulled image
# Warning  Failed     kubelet  Failed to pull image: ... unauthorized
```

---

## Summary Checklist

- [ ] Use private registry for internal images
- [ ] Pin images to specific tags or SHA digests (never `:latest`)
- [ ] Use minimal base images
- [ ] Run as non-root user
- [ ] Scan images in CI/CD with Trivy or Snyk
- [ ] Use `imagePullPolicy: Always` to ensure freshest image
- [ ] Never embed secrets in Dockerfiles or images
