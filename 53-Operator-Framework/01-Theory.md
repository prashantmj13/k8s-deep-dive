# 53 — Operator Framework

## What is the Operator Pattern?

An **Operator** is a method of packaging, deploying, and managing a Kubernetes application using **Custom Resources** and **Custom Controllers**. It encodes operational knowledge (how to install, upgrade, backup, failover) into software.

```
Human Operator knowledge:
 "To upgrade PostgreSQL: drain connections → take backup → upgrade binary → run migrations → verify health"

Kubernetes Operator:
 Watches PostgresCluster CR → detects version change → executes the same procedure automatically
```

---

## Operator Components

```
┌─────────────────────────────────────────┐
│              Operator                   │
│                                         │
│  CRD: PostgresCluster                   │
│  (defines desired state schema)         │
│                                         │
│  Controller: PostgresClusterReconciler  │
│  (watches CRD, manages resources)       │
│                                         │
│  Creates/manages:                       │
│  → StatefulSet (postgres pods)          │
│  → Services (primary, replica)          │
│  → ConfigMaps (postgres.conf)           │
│  → Secrets (passwords)                  │
│  → CronJob (backups)                    │
└─────────────────────────────────────────┘
```

---

## Operator Maturity Model

Operators mature through capability levels:

| Level | Capability |
|-------|-----------|
| 1 — Basic Install | Automated application provisioning |
| 2 — Seamless Upgrades | Patch and minor version upgrades |
| 3 — Full Lifecycle | App lifecycle, storage lifecycle |
| 4 — Deep Insights | Metrics, alerts, log processing |
| 5 — Auto Pilot | Horizontal/vertical scaling, auto-config tuning |

---

## Popular Operators

| Operator | Manages |
|----------|---------|
| cert-manager | TLS certificate lifecycle |
| Prometheus Operator | Prometheus, Alertmanager, Grafana |
| ArgoCD | GitOps deployment |
| Strimzi | Apache Kafka |
| CloudNativePG | PostgreSQL |
| Rook | Ceph storage |
| Jaeger Operator | Distributed tracing |

---

## Operator Frameworks

| Framework | Language | By |
|-----------|----------|-----|
| **Operator SDK** | Go, Ansible, Helm | Red Hat |
| **kubebuilder** | Go | Kubernetes SIG |
| **kopf** | Python | Zalando |
| **Java Operator SDK** | Java | community |
| **shell-operator** | Shell scripts | Flant |

---

## Operator SDK Quickstart (Go)

```bash
# Install operator-sdk
brew install operator-sdk

# Initialize project
operator-sdk init \
  --domain example.com \
  --repo github.com/myorg/myoperator

# Create API
operator-sdk create api \
  --group apps \
  --version v1alpha1 \
  --kind MyApp \
  --resource --controller

# Generate CRD manifests
make manifests generate

# Run locally (against current kubeconfig cluster)
make run

# Build and push image
make docker-build docker-push IMG=registry.example.com/myoperator:v0.1

# Deploy to cluster
make deploy IMG=registry.example.com/myoperator:v0.1
```

---

## Installing Existing Operators

### Via OperatorHub (OLM)

```bash
# Install OLM (Operator Lifecycle Manager)
operator-sdk olm install

# Browse available operators
kubectl get packagemanifests -n olm

# Install cert-manager operator
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Verify
kubectl get pods -n cert-manager
```

### Via Helm

```bash
# Prometheus Operator
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

# cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true
```

---

## Using cert-manager as an Example

Once installed:

```yaml
# Request a certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-tls-cert
  namespace: default
spec:
  secretName: my-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - myapp.example.com
```

The cert-manager operator watches this CR and automatically:
1. Creates a DNS/HTTP challenge
2. Proves domain ownership to Let's Encrypt
3. Stores the signed cert in `my-tls-secret`
4. Renews it before expiry

---

## Operator Lifecycle Manager (OLM)

OLM manages operator installation, upgrades, and dependencies:

```bash
# List installed operators
kubectl get operators
kubectl get csv -A   # ClusterServiceVersion

# Install an operator from OperatorHub
kubectl apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: my-operator
  namespace: operators
spec:
  channel: stable
  name: my-operator
  source: operatorhub
  sourceNamespace: olm
EOF
```
