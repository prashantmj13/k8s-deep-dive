# 36 — TLS in Kubernetes | Exercises

> **Cluster:** kubeadm cluster
> **Estimated time:** 20 minutes

---

## Exercise 1 — List all certificates

```bash
ls /etc/kubernetes/pki/
ls /etc/kubernetes/pki/etcd/
```

---

## Exercise 2 — Inspect the API server certificate

```bash
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A5 "Subject Alternative"
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -subject -issuer
```

---

## Exercise 3 — Check all cert expiry with kubeadm

```bash
kubeadm certs check-expiration
```

---

## Exercise 4 — Inspect the CA certificate

```bash
openssl x509 -in /etc/kubernetes/pki/ca.crt -text -noout | grep -E "Subject:|Issuer:|Not "
```

Note: For the CA, Subject and Issuer should be the same (self-signed).

---

## Exercise 5 — Verify etcd certificates

```bash
openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -noout -subject -dates
openssl x509 -in /etc/kubernetes/pki/etcd/ca.crt -noout -subject
```

---

## Exercise 6 — Check which cert the API server is using

```bash
kubectl describe pod kube-apiserver-controlplane -n kube-system | grep -E "tls-cert|client-ca|etcd-cert"
```

---

## Challenge Questions

1. What happens if the API server certificate expires?
2. What is a Subject Alternative Name (SAN) and why does the API server need multiple ones?
3. Why does etcd have its own separate CA?
4. Where are controller-manager and scheduler credentials stored?
5. How do you renew all certificates at once?
