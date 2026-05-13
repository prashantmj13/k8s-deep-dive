# 36 — TLS in Kubernetes

## Why TLS?

Without TLS, all communication between components is plaintext — anyone on the network can read secrets, tokens, and cluster state. TLS provides:
- **Encryption** — traffic cannot be read in transit
- **Authentication** — each component proves its identity with a certificate
- **Integrity** — data cannot be tampered with

---

## Certificate Types

| Type | Purpose |
|------|---------|
| **CA certificate** | Root of trust — signs all other certs |
| **Server certificate** | Component proves its identity to clients |
| **Client certificate** | Client proves its identity to a server |

---

## Kubernetes CA Hierarchy

Kubernetes uses a self-signed CA. All component certs are signed by this CA.

```
Kubernetes CA (ca.crt / ca.key)
├── kube-apiserver.crt (server cert for API server)
├── apiserver-kubelet-client.crt (API server acts as client to kubelet)
├── apiserver-etcd-client.crt (API server acts as client to etcd)
└── ...

etcd CA (etcd/ca.crt)
├── etcd-server.crt
├── etcd-peer.crt
└── etcd-healthcheck-client.crt

Front-Proxy CA (front-proxy-ca.crt)
└── front-proxy-client.crt
```

---

## Certificate Locations (kubeadm)

```
/etc/kubernetes/pki/
├── ca.crt                          ← Kubernetes CA cert
├── ca.key                          ← Kubernetes CA key (protect!)
├── apiserver.crt                   ← API server server cert
├── apiserver.key
├── apiserver-kubelet-client.crt    ← API server → kubelet client cert
├── apiserver-kubelet-client.key
├── apiserver-etcd-client.crt       ← API server → etcd client cert
├── apiserver-etcd-client.key
├── front-proxy-ca.crt
├── front-proxy-client.crt
├── front-proxy-client.key
├── sa.pub                          ← Service account public key
├── sa.key                          ← Service account private key
└── etcd/
    ├── ca.crt
    ├── server.crt
    ├── server.key
    ├── peer.crt
    ├── peer.key
    └── healthcheck-client.crt
```

---

## Component Certificate Map

| Component | Server Cert | Client Cert (connects to) |
|-----------|-------------|--------------------------|
| kube-apiserver | `apiserver.crt` | `apiserver-etcd-client.crt` (etcd) |
| kubelet | `kubelet.crt` | `kubelet-client.crt` (API server) |
| kube-controller-manager | — | `controller-manager.conf` |
| kube-scheduler | — | `scheduler.conf` |
| etcd | `etcd/server.crt` | `etcd/peer.crt` |

---

## Viewing Certificate Details

```bash
# Decode certificate
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout

# Key fields to check:
# - Subject: CN (common name)
# - Issuer (should be Kubernetes CA)
# - Validity: Not Before / Not After
# - Subject Alternative Names (SANs) — for API server
```

---

## API Server Certificate SANs

The API server cert must include all names/IPs clients use to reach it:

```
X509v3 Subject Alternative Name:
    DNS:controlplane, DNS:kubernetes, DNS:kubernetes.default,
    DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local,
    IP Address:10.96.0.1, IP Address:192.168.1.10
```

---

## Checking for Expiry

```bash
# Check expiry date
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates

# Check all kubeadm-managed certs
kubeadm certs check-expiration
```

---

## Renewing Certificates

```bash
# Renew all certs (kubeadm)
kubeadm certs renew all

# Renew specific cert
kubeadm certs renew apiserver
```

---

## Key Points

- Never share or lose `ca.key` — it can sign any certificate
- Certificates expire (default 1 year for kubeadm)
- API server must be restarted after cert renewal
- etcd has its own separate CA
