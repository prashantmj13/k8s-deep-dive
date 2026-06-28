# 72 — Helm Components and Charts

## Core Helm Concepts

| Term | Meaning |
|------|---------|
| **Chart** | A package of Kubernetes YAML templates + metadata |
| **Release** | A specific deployed instance of a chart (with specific values) |
| **Repository** | An HTTP server hosting one or more charts |
| **Values** | Configuration passed into a chart's templates |
| **Revision** | A version of a release — each `install`/`upgrade`/`rollback` creates one |

---

## Chart Directory Structure

```
mychart/
├── Chart.yaml          # chart metadata (name, version, dependencies)
├── values.yaml          # default configuration values
├── values.schema.json   # optional JSON schema to validate values
├── charts/               # subcharts (dependencies) live here
├── templates/            # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl     # reusable template snippets (named templates)
│   ├── NOTES.txt         # printed after install/upgrade
│   └── tests/            # helm test hooks
├── .helmignore           # files to exclude when packaging
└── README.md
```

---

## Chart.yaml — Metadata

```yaml
apiVersion: v2
name: mychart
description: A Helm chart for my application
type: application          # or "library" for reusable template-only charts
version: 1.2.0              # CHART version (semver, bump on chart changes)
appVersion: "2.5.1"          # APPLICATION version (the app being deployed)
dependencies:
- name: postgresql
  version: "12.x.x"
  repository: "https://charts.bitnami.com/bitnami"
  condition: postgresql.enabled
keywords:
- web
- backend
maintainers:
- name: Platform Team
  email: platform@example.com
```

> **Important distinction:** `version` = chart's own version. `appVersion` = the version of the application it deploys. They are independent.

---

## values.yaml — Default Configuration

```yaml
replicaCount: 2

image:
  repository: myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

ingress:
  enabled: false
  hosts:
  - host: chart-example.local
    paths: ["/"]

postgresql:
  enabled: true
```

---

## Template Example

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        ports:
        - containerPort: 80
```

---

## Built-in Template Objects

| Object | Contains |
|--------|----------|
| `.Values` | Merged values (defaults + overrides) |
| `.Release` | `.Release.Name`, `.Release.Namespace`, `.Release.Revision`, `.Release.IsUpgrade` |
| `.Chart` | Contents of Chart.yaml (`.Chart.Name`, `.Chart.Version`) |
| `.Files` | Access to non-template files in the chart |
| `.Capabilities` | Kubernetes version/API info |

---

## Named Templates (_helpers.tpl)

```
{{/* _helpers.tpl */}}
{{- define "mychart.fullname" -}}
{{- .Release.Name }}-{{ .Chart.Name }}
{{- end }}
```

Used in other templates:
```yaml
metadata:
  name: {{ include "mychart.fullname" . }}
```

---

## Creating a New Chart

```bash
helm create mychart
```

Generates a working starter chart with a sample Deployment, Service, Ingress, HPA, ServiceAccount, and tests — a great learning reference.

---

## Chart Types

| Type | Purpose |
|------|---------|
| `application` (default) | Deploys actual resources |
| `library` | Contains only reusable templates — no resources of its own. Used as a dependency by other charts. |

---

## Packaging and Distributing a Chart

```bash
helm package mychart/                  # creates mychart-1.2.0.tgz
helm lint mychart/                     # validate chart structure
helm template mychart/ | head -50      # render locally without installing

# Push to a repo (using helm-push plugin or OCI registry)
helm push mychart-1.2.0.tgz oci://myregistry.io/charts
```

---

## Dependencies

```bash
# After listing dependencies in Chart.yaml:
helm dependency update mychart/    # downloads deps into charts/
helm dependency list mychart/
helm dependency build mychart/     # rebuild from Chart.lock
```
