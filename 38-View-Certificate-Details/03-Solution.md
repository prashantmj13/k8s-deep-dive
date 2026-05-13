# 38 — View Certificate Details | Solutions

---

## Exercise 1 — API server cert

```
Subject: CN = kube-apiserver
Issuer: CN = kubernetes
Not Before: Jan  1 00:00:00 2026 GMT
Not After : Jan  1 00:00:00 2027 GMT
DNS:controlplane, DNS:kubernetes, DNS:kubernetes.default,
DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local,
IP Address:10.96.0.1, IP Address:192.168.1.10
```

---

## Exercise 4 — Admin kubeconfig CN

```
Subject: CN = kubernetes-admin, O = system:masters
Issuer: CN = kubernetes
Not After : Jan  1 00:00:00 2027 GMT
```

The admin user is `kubernetes-admin` in group `system:masters`.

---

## Exercise 5 — Cert/key match

```
# Both commands should output the same hash:
8f3a9c21b4e5d710a2b3c4d5e6f7a8b9  -
8f3a9c21b4e5d710a2b3c4d5e6f7a8b9  -
```

If hashes differ: the cert was generated with a different key — this will cause TLS handshake failures.

---

## Exercise 7 — etcd cert

```
Subject: CN = controlplane
Issuer: CN = etcd-ca        ← different CA!
Not Before: Jan  1 00:00:00 2026 GMT
Not After : Jan  1 00:00:00 2027 GMT
```

---

## Challenge Answers

**1. Check cert/key are a matching pair?**
```bash
openssl x509 -in cert.crt -noout -modulus | md5sum
openssl rsa  -in cert.key -noout -modulus | md5sum
```
Both must produce the same MD5 hash. The modulus is a mathematical property that links a cert to exactly one private key.

**2. All expiry dates at once?**
```bash
kubeadm certs check-expiration
```

**3. Not After meaning?**
The date after which the certificate is no longer valid. Any TLS connection using the cert after this date will be rejected with `certificate has expired`. Plan renewals at least 30 days before this date.

**4. Admin cert in kubeadm?**
Embedded as base64 in `/etc/kubernetes/admin.conf` under `client-certificate-data`. It's not stored as a separate `.crt` file — it's baked into the kubeconfig.

**5. Different Issuer for etcd?**
etcd uses its own separate CA (`etcd-ca`) for defense in depth. If the main Kubernetes CA were compromised, an attacker still couldn't forge etcd client certificates because they'd need the separate etcd CA key. This limits blast radius.
