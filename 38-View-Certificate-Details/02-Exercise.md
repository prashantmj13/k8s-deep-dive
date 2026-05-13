# 38 — View Certificate Details | Exercises

> **Cluster:** kubeadm cluster
> **Estimated time:** 15 minutes

---

## Exercise 1 — Inspect the API server certificate

```bash
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | \
  grep -E "Subject:|Issuer:|Not Before:|Not After:|DNS:|IP Address:"
```

Note: CN, Issuer, expiry date, and all SANs.

---

## Exercise 2 — Check all cert expiry dates at once

```bash
kubeadm certs check-expiration
```

Which cert expires soonest?

---

## Exercise 3 — List all certs and their expiry

```bash
for cert in $(find /etc/kubernetes/pki -name "*.crt" | sort); do
  echo -n "$cert: "
  openssl x509 -in $cert -noout -enddate 2>/dev/null
done
```

---

## Exercise 4 — Decode and inspect the admin kubeconfig cert

```bash
grep client-certificate-data /etc/kubernetes/admin.conf | \
  awk '{print $2}' | base64 -d | \
  openssl x509 -text -noout | grep -E "Subject:|Issuer:|Not After"
```

What is the CN of the admin user?

---

## Exercise 5 — Verify a cert matches its key

```bash
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -modulus | md5sum
openssl rsa  -in /etc/kubernetes/pki/apiserver.key -noout -modulus | md5sum
```

Both hashes must match — if they differ, the cert and key are mismatched.

---

## Exercise 6 — Live cert from running API server

```bash
openssl s_client \
  -connect 127.0.0.1:6443 \
  -CAfile /etc/kubernetes/pki/ca.crt \
  2>/dev/null | openssl x509 -text -noout | \
  grep -E "Subject:|Not After:|DNS:"
```

---

## Exercise 7 — Check the etcd server cert

```bash
openssl x509 -in /etc/kubernetes/pki/etcd/server.crt \
  -noout -subject -issuer -dates
```

Note: etcd uses its own CA (`etcd/ca.crt`), not the main Kubernetes CA.

---

## Challenge Questions

1. How do you check if a cert and its private key are a matching pair?
2. What kubeadm command shows all cert expiry dates in one view?
3. What does `Not After` mean in a certificate?
4. Where is the admin user's certificate stored in a kubeadm cluster?
5. Why does etcd have a different Issuer than the API server cert?
