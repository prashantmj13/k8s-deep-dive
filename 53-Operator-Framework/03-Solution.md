# 53 — Operator Framework | Solutions

---

## Exercise 2 — cert-manager CRDs

```
certificaterequests.cert-manager.io
certificates.cert-manager.io
challenges.acme.cert-manager.io
clusterissuers.cert-manager.io
issuers.cert-manager.io
orders.acme.cert-manager.io
```

Six CRDs — each a domain concept cert-manager manages.

---

## Exercise 4 — Certificate status

```bash
kubectl get certificate test-cert
```
```
NAME        READY   SECRET             AGE
test-cert   True    test-cert-secret   15s
```

`READY: True` means the cert was successfully issued and stored in the secret.

---

## Exercise 5 — Cert details

```
Subject: CN = test.example.com
Not Before: May 13 00:00:00 2026 GMT
Not After : May 14 00:00:00 2026 GMT   ← 24h duration as specified
```

---

## Challenge Answers

**1. Operator vs Helm chart?**
- Helm chart: installs static Kubernetes resources (templates). No ongoing management after install. Can't react to events, failures, or state changes.
- Operator: includes a running controller that continuously watches and manages the application. Can handle upgrades, backups, failover, scaling — all automatically. Encodes operational knowledge in code, not just templates.

**2. OLM (Operator Lifecycle Manager)?**
OLM is a system for managing operator installation, upgrades, and dependencies in a cluster. It provides: a catalog of available operators (OperatorHub), dependency resolution between operators, controlled upgrade channels, and ClusterServiceVersion (CSV) objects that describe what an operator installs. Think of it as apt/yum but for operators.

**3. Five maturity levels?**
1. Basic Install — automated provisioning
2. Seamless Upgrades — patch and minor upgrades
3. Full Lifecycle — backup, restore, failure recovery
4. Deep Insights — metrics, alerting, log management
5. Auto Pilot — auto-scaling, auto-configuration, auto-healing

**4. kopf vs operator-sdk?**
- kopf (Python): faster to prototype, great for teams already in Python, simpler for straightforward use cases, less boilerplate. Production-ready but smaller ecosystem.
- operator-sdk (Go): better performance, native Kubernetes tooling (kubebuilder), access to all client-go features, larger community, standard choice for production operators. More verbose but more powerful.

**5. What cert-manager did automatically?**
1. Watched for the new Certificate CR
2. Created a CertificateRequest object
3. Called the selfSigned ClusterIssuer to sign it
4. Received the signed certificate
5. Created the `test-cert-secret` Secret with `tls.crt`, `tls.key`, and `ca.crt`
6. Updated the Certificate CR status to `Ready: True`
7. Scheduled a renewal job for 1 hour before expiry (renewBefore: 1h)

All of this happened within seconds, automatically — this is the power of the Operator pattern.
