# 39 — Certificates API | Solutions

---

## Exercise 2 — CSR submitted

```bash
kubectl get csr
```
```
NAME   AGE   SIGNERNAME                            REQUESTOR          CONDITION
bob    5s    kubernetes.io/kube-apiserver-client   kubernetes-admin   Pending
```

---

## Exercise 3 — After approval

```
NAME   AGE   CONDITION
bob    30s   Approved,Issued
```

---

## Exercise 4 — Certificate details

```
Subject: CN = bob, O = developers
Issuer: CN = kubernetes
Not Before: May 13 00:00:00 2026 GMT
Not After : May 14 00:00:00 2026 GMT   ← 24h because expirationSeconds=86400
```

---

## Exercise 5 — Bob's access

```bash
kubectl get pods
```
```
Error from server (Forbidden): pods is forbidden: User "bob" cannot list resource "pods"
```

Auth succeeded (cert valid), authorization failed (no RBAC). Correct expected behaviour.

---

## Exercise 7 — Denied CSR

```bash
kubectl get csr eve
```
```
NAME   AGE   CONDITION
eve    10s   Denied
```

The cert is never issued. `status.certificate` remains empty.

---

## Challenge Answers

**1. What component signs after approval?**
The **kube-controller-manager** performs the actual signing. It watches for `Approved` CSRs and signs them using the cluster CA key specified by `--cluster-signing-cert-file` and `--cluster-signing-key-file`.

**2. signerName field?**
Specifies which signer should process the request. Different signers produce certs for different purposes. `kubernetes.io/kube-apiserver-client` produces certs trusted for authenticating to the API server. Correct signerName is required — using the wrong one results in a cert that may not work for your intended purpose.

**3. After a CSR is denied?**
The CSR status shows `Denied`. The certificate is never issued. The CSR object remains until deleted or it expires. The user must generate a new CSR if they want to try again.

**4. Default validity?**
Controlled by `expirationSeconds` in the CSR spec. If not set, the default is the minimum of the signer's max duration. For kubeadm clusters the cluster signing duration defaults to 1 year. In exercises we set 86400 (24h) explicitly.

**5. Can user submit their own CSR?**
Yes — any authenticated user can create a CSR object if they have permission. By default users have permission to create CSRs for themselves. However, only admins (users with `certificates approve` permission) can approve them. This separation ensures users can request but not self-approve certs.
