# 53 — Operator Framework | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 25 minutes

---

## Exercise 1 — Install cert-manager operator

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Wait for it to be ready
kubectl wait --for=condition=Available deployment/cert-manager -n cert-manager --timeout=120s
kubectl get pods -n cert-manager
```

---

## Exercise 2 — List cert-manager CRDs

```bash
kubectl get crds | grep cert-manager
```

> Operators install their own CRDs — you should see: certificates, issuers, clusterissuers, certificaterequests, orders, challenges.

---

## Exercise 3 — Create a self-signed ClusterIssuer

```yaml
# selfsigned-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

```bash
kubectl apply -f selfsigned-issuer.yaml
kubectl get clusterissuer
kubectl describe clusterissuer selfsigned-issuer
```

---

## Exercise 4 — Request a certificate

```yaml
# test-cert.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: test-cert
  namespace: default
spec:
  secretName: test-cert-secret
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  dnsNames:
  - test.example.com
  duration: 24h
  renewBefore: 1h
```

```bash
kubectl apply -f test-cert.yaml
kubectl get certificate test-cert
kubectl describe certificate test-cert
```

---

## Exercise 5 — Verify the cert was issued

```bash
kubectl get secret test-cert-secret
kubectl get secret test-cert-secret -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -subject -dates
```

The cert-manager operator automatically: watched the Certificate CR → requested signing from the ClusterIssuer → stored the result in a Secret.

---

## Exercise 6 — Observe operator reconciliation

```bash
# Watch cert-manager controller logs
kubectl logs -n cert-manager deployment/cert-manager -f | grep -E "cert|reconcil" | head -20
```

---

## Exercise 7 — Cleanup

```bash
kubectl delete certificate test-cert
kubectl delete clusterissuer selfsigned-issuer
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

---

## Challenge Questions

1. What is the difference between an Operator and a Helm chart?
2. What is OLM (Operator Lifecycle Manager)?
3. What are the 5 maturity levels of the Operator Capability Model?
4. Why would you use kopf (Python) vs operator-sdk (Go)?
5. What did cert-manager automatically do when you created the Certificate CR?
