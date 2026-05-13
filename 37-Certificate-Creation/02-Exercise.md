# 37 — Certificate Creation | Exercises

> **Cluster:** kubeadm cluster or any Linux machine with openssl
> **Estimated time:** 20 minutes

---

## Exercise 1 — Generate a CA

```bash
mkdir -p /tmp/certs && cd /tmp/certs

openssl genrsa -out ca.key 2048
openssl req -new -x509 -days 3650 \
  -key ca.key \
  -subj "/CN=MY-K8S-CA" \
  -out ca.crt

openssl x509 -in ca.crt -noout -subject -issuer -dates
```

> **Observe:** Subject and Issuer are the same — self-signed.

---

## Exercise 2 — Generate admin user certificate

```bash
openssl genrsa -out admin.key 2048

openssl req -new \
  -key admin.key \
  -subj "/CN=kube-admin/O=system:masters" \
  -out admin.csr

openssl x509 -req \
  -in admin.csr \
  -CA ca.crt -CAkey ca.key \
  -CAcreateserial \
  -out admin.crt -days 365

openssl x509 -in admin.crt -noout -subject -issuer
```

---

## Exercise 3 — Generate a developer user certificate

Create a cert for user `dev1` in group `developers`:

```bash
openssl genrsa -out dev1.key 2048

openssl req -new \
  -key dev1.key \
  -subj "/CN=dev1/O=developers" \
  -out dev1.csr

openssl x509 -req \
  -in dev1.csr \
  -CA ca.crt -CAkey ca.key \
  -CAcreateserial \
  -out dev1.crt -days 365

openssl x509 -in dev1.crt -noout -subject
```

---

## Exercise 4 — Generate API server cert with SANs

```bash
cat > /tmp/certs/apiserver-san.cnf << EOF
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
IP.1 = 10.96.0.1
IP.2 = 127.0.0.1
EOF

openssl genrsa -out apiserver.key 2048

openssl req -new \
  -key apiserver.key \
  -subj "/CN=kube-apiserver" \
  -config apiserver-san.cnf \
  -out apiserver.csr

openssl x509 -req \
  -in apiserver.csr \
  -CA ca.crt -CAkey ca.key \
  -CAcreateserial \
  -extensions v3_req \
  -extfile apiserver-san.cnf \
  -out apiserver.crt -days 365

openssl x509 -in apiserver.crt -text -noout | grep -A6 "Subject Alternative"
```

---

## Exercise 5 — Verify all certs against the CA

```bash
openssl verify -CAfile ca.crt admin.crt
openssl verify -CAfile ca.crt dev1.crt
openssl verify -CAfile ca.crt apiserver.crt
```

---

## Exercise 6 — Add admin cert to kubeconfig

```bash
kubectl config set-credentials my-admin \
  --client-certificate=/tmp/certs/admin.crt \
  --client-key=/tmp/certs/admin.key

kubectl config view | grep my-admin
```

---

## Challenge Questions

1. What fields in the CSR become the Kubernetes username and group?
2. Why does the API server certificate need Subject Alternative Names?
3. What does `CAcreateserial` do?
4. What group must a user belong to for cluster-admin access?
5. How do you verify a certificate was signed by a specific CA?
