# 39 — Certificates API

## What is the Certificates API?

The Kubernetes Certificates API (`certificates.k8s.io`) provides a **built-in workflow for requesting and approving TLS certificates** without needing direct access to the CA key. This is the recommended way to onboard new users.

---

## The CSR Workflow

```
User generates key+CSR
        ↓
Admin creates CertificateSigningRequest object in Kubernetes
        ↓
Admin (or automated controller) approves/denies the CSR
        ↓
Controller Manager signs the cert using the cluster CA
        ↓
User retrieves the signed cert from the CSR object
```

---

## Step 1 — User generates key and CSR

```bash
openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -subj "/CN=jane/O=developers" -out jane.csr
```

---

## Step 2 — Create the CertificateSigningRequest object

```bash
# Base64 encode the CSR (no line breaks)
CSR=$(cat jane.csr | base64 -w 0)
```

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: <base64-csr-here>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400   # 24 hours
  usages:
  - client auth
```

```bash
kubectl apply -f jane-csr.yaml
kubectl get csr
```

Output:
```
NAME   AGE   SIGNERNAME                            REQUESTOR   CONDITION
jane   10s   kubernetes.io/kube-apiserver-client   admin       Pending
```

---

## Step 3 — Approve or Deny

```bash
# Approve
kubectl certificate approve jane

# Deny
kubectl certificate deny jane

# Check status
kubectl get csr jane
```

After approval:
```
NAME   AGE   CONDITION
jane   30s   Approved,Issued
```

---

## Step 4 — Retrieve the Signed Certificate

```bash
kubectl get csr jane -o jsonpath='{.status.certificate}' | base64 -d > jane.crt

# Verify
openssl x509 -in jane.crt -noout -subject -issuer
```

---

## Signer Names

| Signer | Purpose |
|--------|---------|
| `kubernetes.io/kube-apiserver-client` | Client certs for API server auth |
| `kubernetes.io/kube-apiserver-client-kubelet` | Kubelet client certs |
| `kubernetes.io/kubelet-serving` | Kubelet server certs |
| `kubernetes.io/legacy-unknown` | Legacy, avoid in new clusters |

---

## Who Approves CSRs?

By default, **human admins** must approve CSRs manually. Automated approval is handled by:
- The kubelet bootstrap controller (auto-approves kubelet CSRs)
- Custom controllers you write for your use case

---

## CSR Lifecycle

| Status | Meaning |
|--------|---------|
| `Pending` | Waiting for approval |
| `Approved` | Approved but not yet signed |
| `Approved,Issued` | Signed cert available |
| `Denied` | Request was rejected |
| `Failed` | Signing failed |

---

## Clean Up Old CSRs

```bash
kubectl delete csr jane
# or let them expire (default TTL: 1 hour after approval/denial)
```

---

## Controller Manager Role

The **kube-controller-manager** performs the actual signing using:
- `--cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt`
- `--cluster-signing-key-file=/etc/kubernetes/pki/ca.key`

If these flags are missing, CSR signing won't work.
