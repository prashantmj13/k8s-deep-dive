# 38 — View Certificate Details

## Why View Certificates?

You need to inspect certificates to:
- Debug TLS connection failures
- Check expiry dates before they cause cluster outages
- Verify correct CN/O values for authentication
- Confirm SANs cover all required endpoints
- Troubleshoot "x509: certificate signed by unknown authority" errors

---

## Core Tool: openssl x509

```bash
# Full human-readable output
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout

# Just subject and issuer
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -subject -issuer

# Just validity dates
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates

# Just SANs
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -ext subjectAltName

# Serial number
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -serial
```

---

## Key Fields to Inspect

```
Certificate:
    Data:
        Version: 3
        Serial Number: 1234 (unique per CA)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = kubernetes          ← who signed it
        Validity
            Not Before: Jan  1 00:00:00 2026 GMT
            Not After : Jan  1 00:00:00 2027 GMT   ← expiry!
        Subject: CN = kube-apiserver    ← component identity
        X509v3 Extensions:
            X509v3 Subject Alternative Name:
                DNS:kubernetes, DNS:kubernetes.default, ...
            X509v3 Key Usage:
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
```

---

## Check All kubeadm Certs at Once

```bash
kubeadm certs check-expiration
```

Output:
```
CERTIFICATE                EXPIRES                  RESIDUAL TIME   AUTHORITY
admin.conf                 Jan 01, 2027 00:00 UTC   364d            ca
apiserver                  Jan 01, 2027 00:00 UTC   364d            ca
apiserver-etcd-client      Jan 01, 2027 00:00 UTC   364d            etcd-ca
apiserver-kubelet-client   Jan 01, 2027 00:00 UTC   364d            ca
controller-manager.conf    Jan 01, 2027 00:00 UTC   364d            ca
etcd-healthcheck-client    Jan 01, 2027 00:00 UTC   364d            etcd-ca
etcd-peer                  Jan 01, 2027 00:00 UTC   364d            etcd-ca
etcd-server                Jan 01, 2027 00:00 UTC   364d            etcd-ca
front-proxy-client         Jan 01, 2027 00:00 UTC   364d            front-proxy-ca
scheduler.conf             Jan 01, 2027 00:00 UTC   364d            ca
```

---

## Inspect Certs Embedded in kubeconfig

kubeconfig files contain base64-encoded certs:

```bash
# Extract and decode the client cert from admin.conf
grep client-certificate-data /etc/kubernetes/admin.conf | \
  awk '{print $2}' | base64 -d | \
  openssl x509 -text -noout | grep -E "Subject:|Issuer:|Not "
```

---

## Inspect Cert from a Running Pod/Component

```bash
# Get the cert path from the API server pod
kubectl describe pod kube-apiserver-controlplane -n kube-system | grep tls-cert-file

# Connect to API server and view its presented cert
openssl s_client -connect 127.0.0.1:6443 \
  -CAfile /etc/kubernetes/pki/ca.crt 2>/dev/null | \
  openssl x509 -text -noout | grep -E "Subject:|Not After"
```

---

## Common Certificate Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `x509: certificate has expired` | Cert past expiry date | `kubeadm certs renew all` |
| `x509: certificate signed by unknown authority` | Wrong CA or CA mismatch | Verify `--certificate-authority` flag |
| `x509: certificate is valid for X, not Y` | Missing SAN | Regenerate cert with correct SANs |
| `tls: bad certificate` | Wrong cert/key pair | Verify cert and key match |

### Verify cert and key match

```bash
# These should output the same modulus hash
openssl x509 -in apiserver.crt -noout -modulus | md5sum
openssl rsa  -in apiserver.key -noout -modulus | md5sum
```

---

## Automated Expiry Check Script

```bash
#!/bin/bash
PKI=/etc/kubernetes/pki
for cert in $(find $PKI -name "*.crt"); do
  expiry=$(openssl x509 -in $cert -noout -enddate 2>/dev/null | cut -d= -f2)
  echo "$cert: $expiry"
done
```
