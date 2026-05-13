# 40 — KubeConfig

## What is KubeConfig?

KubeConfig is the configuration file that `kubectl` uses to know **which cluster to talk to**, **how to authenticate**, and **which namespace to use by default**. Default location: `~/.kube/config`.

---

## KubeConfig Structure

A kubeconfig file has three key sections:

```yaml
apiVersion: v1
kind: Config
current-context: dev-user@my-cluster

clusters:                          # WHERE to connect
- name: my-cluster
  cluster:
    server: https://192.168.1.10:6443
    certificate-authority: /etc/kubernetes/pki/ca.crt
    # OR inline: certificate-authority-data: <base64>

users:                             # WHO you are
- name: dev-user
  user:
    client-certificate: /path/to/dev.crt
    client-key: /path/to/dev.key
    # OR inline:
    # client-certificate-data: <base64>
    # client-key-data: <base64>
    # OR token:
    # token: eyJhbGci...

contexts:                          # WHICH cluster + user + namespace
- name: dev-user@my-cluster
  context:
    cluster: my-cluster
    user: dev-user
    namespace: development         # optional default namespace
```

---

## kubectl config Commands

```bash
# View full kubeconfig
kubectl config view

# View with secrets shown (decode base64)
kubectl config view --raw

# Current context
kubectl config current-context

# List all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context dev-user@my-cluster

# Set default namespace for current context
kubectl config set-context --current --namespace=development

# Add a new cluster
kubectl config set-cluster my-cluster \
  --server=https://192.168.1.10:6443 \
  --certificate-authority=/etc/kubernetes/pki/ca.crt

# Add a new user
kubectl config set-credentials dev-user \
  --client-certificate=/path/to/dev.crt \
  --client-key=/path/to/dev.key

# Add a new context
kubectl config set-context dev-user@my-cluster \
  --cluster=my-cluster \
  --user=dev-user \
  --namespace=development

# Delete a context
kubectl config delete-context dev-user@my-cluster
```

---

## Multiple Kubeconfig Files

You can merge multiple kubeconfig files using the `KUBECONFIG` environment variable:

```bash
export KUBECONFIG=~/.kube/config:~/.kube/cluster2-config:~/.kube/cluster3-config

# View merged config
kubectl config view

# Permanently merge into one file
kubectl config view --flatten > ~/.kube/merged-config
```

---

## Inline vs File-based Certs

| Method | Field | Pros/Cons |
|--------|-------|-----------|
| File path | `certificate-authority` | Easy to update cert without editing kubeconfig |
| Inline base64 | `certificate-authority-data` | Single portable file, no external dependencies |

Convert file to inline:
```bash
cat /etc/kubernetes/pki/ca.crt | base64 -w 0
# Paste output into certificate-authority-data
```

---

## Example: Multi-Cluster KubeConfig

```yaml
apiVersion: v1
kind: Config
current-context: admin@production

clusters:
- name: production
  cluster:
    server: https://prod.example.com:6443
    certificate-authority-data: <base64-prod-ca>
- name: staging
  cluster:
    server: https://staging.example.com:6443
    certificate-authority-data: <base64-staging-ca>

users:
- name: admin
  user:
    client-certificate-data: <base64-cert>
    client-key-data: <base64-key>
- name: readonly
  user:
    token: eyJhbGci...

contexts:
- name: admin@production
  context:
    cluster: production
    user: admin
    namespace: default
- name: readonly@staging
  context:
    cluster: staging
    user: readonly
    namespace: monitoring
```

---

## Override KubeConfig per Command

```bash
# Use a different kubeconfig file
kubectl get pods --kubeconfig=/path/to/other-config

# Use a different context
kubectl get pods --context=staging-context

# Use a different namespace
kubectl get pods -n kube-system
```
