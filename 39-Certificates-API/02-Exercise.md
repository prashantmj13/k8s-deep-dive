# 39 — Certificates API | Exercises

> **Cluster:** kubeadm cluster
> **Estimated time:** 20 minutes

---

## Exercise 1 — Generate key and CSR for a new user

```bash
openssl genrsa -out /tmp/bob.key 2048
openssl req -new \
  -key /tmp/bob.key \
  -subj "/CN=bob/O=developers" \
  -out /tmp/bob.csr
```

---

## Exercise 2 — Submit a CertificateSigningRequest

```bash
CSR=$(cat /tmp/bob.csr | base64 -w 0)

cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: bob
spec:
  request: ${CSR}
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
  - client auth
EOF

kubectl get csr
```

---

## Exercise 3 — Approve the CSR

```bash
kubectl certificate approve bob
kubectl get csr bob
```

---

## Exercise 4 — Retrieve and verify the signed certificate

```bash
kubectl get csr bob -o jsonpath='{.status.certificate}' | base64 -d > /tmp/bob.crt

openssl x509 -in /tmp/bob.crt -noout -subject -issuer -dates
```

---

## Exercise 5 — Add bob to kubeconfig and test access

```bash
kubectl config set-credentials bob \
  --client-certificate=/tmp/bob.crt \
  --client-key=/tmp/bob.key

kubectl config set-context bob-context \
  --cluster=kubernetes \
  --user=bob

kubectl config use-context bob-context
kubectl get pods
# Expected: Forbidden (no RBAC yet)

kubectl config use-context kubernetes-admin@kubernetes
```

---

## Exercise 6 — View the CSR object details

```bash
kubectl describe csr bob
kubectl get csr bob -o yaml | grep -E "status|condition|certificate" | head -15
```

---

## Exercise 7 — Deny a CSR

```bash
# Create another CSR
openssl genrsa -out /tmp/eve.key 2048
openssl req -new -key /tmp/eve.key -subj "/CN=eve" -out /tmp/eve.csr
CSR2=$(cat /tmp/eve.csr | base64 -w 0)

cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: eve
spec:
  request: ${CSR2}
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
  - client auth
EOF

kubectl certificate deny eve
kubectl get csr eve
```

---

## Exercise 8 — Cleanup

```bash
kubectl delete csr bob eve
```

---

## Challenge Questions

1. What component signs the certificate after approval?
2. What is the `signerName` field used for?
3. What happens to a CSR after it is denied?
4. How long is a signed certificate valid by default?
5. Can a user submit their own CSR or does an admin need to do it?
