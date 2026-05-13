# 37 — TLS Certificate Creation

## Overview

Every component in Kubernetes needs a TLS certificate to communicate securely. You can generate them manually with `openssl` or let `kubeadm` handle it automatically.

---

## What You Need

| Item | Description |
|------|-------------|
| CA key + cert | The root of trust — signs all component certs |
| Private key | Per-component secret key |
| CSR | Certificate Signing Request — describes what you want |
| Signed cert | CSR signed by the CA |

---

## Step 1 — Generate the CA (self-signed)

```bash
# Generate CA private key
openssl genrsa -out ca.key 2048

# Generate CA self-signed certificate (10 year validity)
openssl req -new -x509 -days 3650 \
  -key ca.key \
  -subj "/CN=KUBERNETES-CA" \
  -out ca.crt
```

The CA cert (`ca.crt`) and key (`ca.key`) are the root of trust. All other certs must be signed by this CA.

---

## Step 2 — Generate a Component Certificate

Example: generating the admin user certificate

```bash
# 1. Generate private key
openssl genrsa -out admin.key 2048

# 2. Create CSR — CN is the username, O is the group
#    system:masters group gives cluster-admin access
openssl req -new \
  -key admin.key \
  -subj "/CN=kube-admin/O=system:masters" \
  -out admin.csr

# 3. Sign with CA
openssl x509 -req \
  -in admin.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out admin.crt \
  -days 365
```

---

## Certificate for the API Server (needs SANs)

The API server cert must include all DNS names and IPs used to reach it:

```bash
# Create a config file for SANs
cat > apiserver-san.cnf << EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
DNS.5 = controlplane
IP.1 = 10.96.0.1
IP.2 = 192.168.1.10
EOF

# Generate key
openssl genrsa -out apiserver.key 2048

# Generate CSR with SANs
openssl req -new \
  -key apiserver.key \
  -subj "/CN=kube-apiserver" \
  -config apiserver-san.cnf \
  -out apiserver.csr

# Sign with CA
openssl x509 -req \
  -in apiserver.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -extensions v3_req \
  -extfile apiserver-san.cnf \
  -out apiserver.crt \
  -days 365
```

---

## Common Component Certificates

| Component | CN | O (Group) |
|-----------|-----|-----------|
| admin user | `kube-admin` | `system:masters` |
| kube-controller-manager | `system:kube-controller-manager` | — |
| kube-scheduler | `system:kube-scheduler` | — |
| kube-proxy | `system:kube-proxy` | — |
| kubelet | `system:node:<nodename>` | `system:nodes` |
| kube-apiserver | `kube-apiserver` | — (needs SANs) |
| etcd server | `etcd-server` | — |

---

## Using the Certificate with kubectl

```bash
# Method 1: Pass as flags
kubectl get pods \
  --server=https://192.168.1.10:6443 \
  --certificate-authority=ca.crt \
  --client-certificate=admin.crt \
  --client-key=admin.key

# Method 2: Embed in kubeconfig
kubectl config set-credentials admin \
  --client-certificate=admin.crt \
  --client-key=admin.key

# Method 3: Embed as base64 in kubeconfig YAML
cat admin.crt | base64 -w 0
```

---

## Verify a Certificate

```bash
# Check CN, O, SANs, validity
openssl x509 -in admin.crt -text -noout

# Verify cert was signed by your CA
openssl verify -CAfile ca.crt admin.crt
# Output: admin.crt: OK
```

---

## kubeadm Generates All Certs Automatically

On a kubeadm cluster all certs are auto-generated at `kubeadm init`:
```
/etc/kubernetes/pki/
```

You only need manual cert creation when:
- Creating user certificates (kubeadm doesn't manage users)
- Replacing expired certs manually
- Setting up a cluster without kubeadm
