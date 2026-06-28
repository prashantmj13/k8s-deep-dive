# 71 — Helm: Introduction and Installation

## What is Helm?

Helm is the **package manager for Kubernetes** — think `apt`/`yum`/`npm` but for Kubernetes manifests. It packages a set of YAML templates (a **chart**) into a reusable, versioned, configurable unit.

---

## Why Helm?

| Without Helm | With Helm |
|---------------|-----------|
| Maintain dozens of raw YAML files per app | One chart with templated values |
| Manual find/replace for different environments | `values.yaml` per environment |
| No versioning of "what's deployed" | `helm history`, rollback built-in |
| Copy-paste boilerplate across services | Reusable chart templates and subcharts |
| Manual dependency ordering | Helm manages chart dependencies |

---

## Helm Architecture (v3)

Helm 3 is **client-only** — no server-side component (unlike Helm 2's Tiller, which was removed for security reasons).

```
helm CLI  →  reads chart + values  →  renders templates  →  kubectl apply equivalent  →  API server
                                                                                              ↓
                                                                                    Release tracked as
                                                                                    a Secret in-cluster
```

Release state (what's installed, with what values, at what revision) is stored as **Secrets** in the target namespace — no separate Tiller pod needed.

---

## Installing Helm

```bash
# Linux/macOS via script
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# macOS via brew
brew install helm

# Linux via apt (Debian/Ubuntu)
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update && sudo apt-get install helm

# Verify
helm version
```

---

## Helm Repositories

A repository is an HTTP server hosting packaged charts (`.tgz`) and an `index.yaml`.

```bash
# Add a repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Update local repo cache
helm repo update

# List added repos
helm repo list

# Search for a chart
helm search repo nginx
helm search hub wordpress    # searches Artifact Hub
```

---

## First Install

```bash
# Search and inspect a chart before installing
helm show chart bitnami/nginx
helm show values bitnami/nginx | head -30

# Install a release
helm install my-nginx bitnami/nginx

# List installed releases
helm list
helm list -A    # all namespaces
```

---

## Helm Client Configuration

```bash
# Config locations (XDG-compliant)
~/.config/helm/      # repositories.yaml, registry config
~/.cache/helm/        # cached chart downloads

# Set a custom kubeconfig context
helm install myapp ./mychart --kube-context my-cluster

# Set namespace
helm install myapp ./mychart -n production --create-namespace
```

---

## Helm Plugins

```bash
helm plugin list
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade myapp ./mychart    # preview changes before applying
```

---

## Verifying Installation

```bash
helm version --short
kubectl get secrets -l owner=helm    # see release secrets stored in cluster
helm env                              # show Helm's environment variables
```
