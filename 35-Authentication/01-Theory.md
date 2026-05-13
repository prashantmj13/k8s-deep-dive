# 35 — Authentication in Kubernetes

## Who Can Access the Cluster?

Kubernetes has two types of accounts:

| Type | Managed by | Used by |
|------|-----------|---------|
| **User Accounts** | External (certs, OIDC) | Humans, kubectl |
| **Service Accounts** | Kubernetes | Pods, controllers |

Kubernetes does NOT have a built-in user database. Users are authenticated via external mechanisms.

---

## Authentication Methods

### 1. X.509 Client Certificates (most common)

Users present a TLS client certificate. The `CN` field becomes the username, `O` field becomes the group.

```bash
# Generate private key
openssl genrsa -out jane.key 2048

# Create CSR with CN=jane, O=developers
openssl req -new -key jane.key \
  -subj "/CN=jane/O=developers" \
  -out jane.csr

# Sign with cluster CA (admin action)
openssl x509 -req -in jane.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial -out jane.crt -days 365

# Configure kubectl to use the cert
kubectl config set-credentials jane \
  --client-certificate=jane.crt \
  --client-key=jane.key
```

### 2. Service Account Tokens

Pods automatically get a service account token mounted at:
```
/var/run/secrets/kubernetes.io/serviceaccount/token
```

Used to authenticate pod requests to the API server.

### 3. OIDC Tokens

Integrate with external identity providers (Google, Azure AD, Okta):
```
API server flag: --oidc-issuer-url=https://accounts.google.com
```

### 4. Static Token File (deprecated)

```bash
# token-auth.csv format: token,user,uid,group
abc123token,jane,u0001,developers
```
```
kube-apiserver: --token-auth-file=token-auth.csv
```

---

## Certificates API — CSR Workflow

Kubernetes has a built-in API for certificate signing:

### Step 1: User generates key and CSR
```bash
openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -subj "/CN=jane/O=developers" -out jane.csr
```

### Step 2: Admin creates CertificateSigningRequest object
```bash
cat jane.csr | base64 -w 0
```

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: <base64-encoded-csr>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
  - client auth
```

### Step 3: Admin approves
```bash
kubectl certificate approve jane
```

### Step 4: User retrieves certificate
```bash
kubectl get csr jane -o jsonpath='{.status.certificate}' | base64 -d > jane.crt
```

---

## kubeconfig — Storing Auth Info

kubeconfig file (`~/.kube/config`) stores:
- **Clusters** — API server endpoints and CA certs
- **Users** — credentials (certs, tokens)
- **Contexts** — cluster + user combinations

```yaml
apiVersion: v1
kind: Config
clusters:
- name: my-cluster
  cluster:
    server: https://192.168.1.10:6443
    certificate-authority: /etc/kubernetes/pki/ca.crt
users:
- name: jane
  user:
    client-certificate: jane.crt
    client-key: jane.key
contexts:
- name: jane@my-cluster
  context:
    cluster: my-cluster
    user: jane
current-context: jane@my-cluster
```

```bash
# Switch context
kubectl config use-context jane@my-cluster

# View current context
kubectl config current-context

# List contexts
kubectl config get-contexts
```

---

## Service Accounts Deep Dive

```bash
# Create service account
kubectl create serviceaccount myapp-sa

# Use in a pod
spec:
  serviceAccountName: myapp-sa

# List service accounts
kubectl get serviceaccounts

# Default SA is automatically assigned if none specified
# Token is mounted at /var/run/secrets/kubernetes.io/serviceaccount/
```
