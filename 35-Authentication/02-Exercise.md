# 35 — Authentication | Exercises

> **Cluster:** kubeadm cluster
> **Estimated time:** 25 minutes

---

## Exercise 1 — Inspect the current kubeconfig

```bash
kubectl config view
kubectl config current-context
kubectl config get-contexts
kubectl config get-clusters
kubectl config get-users
```

---

## Exercise 2 — Generate a user certificate (manual method)

```bash
# Generate private key
openssl genrsa -out /tmp/jane.key 2048

# Create CSR
openssl req -new -key /tmp/jane.key \
  -subj "/CN=jane/O=developers" \
  -out /tmp/jane.csr

# Sign with cluster CA
openssl x509 -req -in /tmp/jane.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out /tmp/jane.crt \
  -days 365

ls /tmp/jane.*
```

---

## Exercise 3 — Add user to kubeconfig

```bash
kubectl config set-credentials jane \
  --client-certificate=/tmp/jane.crt \
  --client-key=/tmp/jane.key

kubectl config set-context jane-context \
  --cluster=kubernetes \
  --user=jane

kubectl config get-contexts
```

---

## Exercise 4 — Use the Certificates API (CSR workflow)

```bash
# Encode CSR
CSR=$(cat /tmp/jane.csr | base64 -w 0)

# Create CSR object
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane-csr
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

## Exercise 5 — Approve and retrieve certificate

```bash
kubectl certificate approve jane-csr
kubectl get csr jane-csr

# Retrieve signed certificate
kubectl get csr jane-csr -o jsonpath='{.status.certificate}' | base64 -d > /tmp/jane-signed.crt
cat /tmp/jane-signed.crt | openssl x509 -text | grep -E "Subject:|Issuer:"
```

---

## Exercise 6 — Test access as jane (expect forbidden)

```bash
kubectl config use-context jane-context
kubectl get pods
# Expected: Forbidden — jane has no RBAC permissions yet
```

Switch back to admin:
```bash
kubectl config use-context kubernetes-admin@kubernetes
```

---

## Exercise 7 — Service Account

```bash
kubectl create serviceaccount app-sa
kubectl get serviceaccounts
kubectl describe serviceaccount app-sa

# Create pod using this SA
kubectl run sa-test --image=nginx --overrides='{"spec":{"serviceAccountName":"app-sa"}}'
kubectl exec sa-test -- cat /var/run/secrets/kubernetes.io/serviceaccount/token | cut -c1-50
```

---

## Challenge Questions

1. What is the CN field in a client certificate used for in Kubernetes?
2. What is the O field used for?
3. Can you create a Kubernetes user with `kubectl`?
4. Where is the kubeconfig file located by default?
5. What happens if a pod does not specify a `serviceAccountName`?
