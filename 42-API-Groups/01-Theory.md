# 42 — API Groups

## What are API Groups?

Kubernetes organizes its API into **groups** to allow independent versioning and extension of different resource types. When you write a YAML manifest, the `apiVersion` field specifies both the group and version.

---

## apiVersion Format

```
apiVersion: <group>/<version>
```

| apiVersion | Group | Version |
|-----------|-------|---------|
| `v1` | core (no group name) | v1 |
| `apps/v1` | apps | v1 |
| `batch/v1` | batch | v1 |
| `autoscaling/v2` | autoscaling | v2 |
| `rbac.authorization.k8s.io/v1` | rbac.authorization.k8s.io | v1 |
| `networking.k8s.io/v1` | networking.k8s.io | v1 |
| `certificates.k8s.io/v1` | certificates.k8s.io | v1 |
| `storage.k8s.io/v1` | storage.k8s.io | v1 |

---

## Core API Group (v1)

Resources with no group prefix — the original Kubernetes objects:

```
pods, services, configmaps, secrets, namespaces,
nodes, persistentvolumes, persistentvolumeclaims,
endpoints, events, replicationcontrollers,
serviceaccounts, resourcequotas, limitranges
```

---

## Named API Groups

| Group | Key Resources |
|-------|--------------|
| `apps` | Deployment, ReplicaSet, DaemonSet, StatefulSet |
| `batch` | Job, CronJob |
| `autoscaling` | HorizontalPodAutoscaler |
| `rbac.authorization.k8s.io` | Role, ClusterRole, RoleBinding, ClusterRoleBinding |
| `networking.k8s.io` | Ingress, NetworkPolicy, IngressClass |
| `storage.k8s.io` | StorageClass, VolumeAttachment |
| `certificates.k8s.io` | CertificateSigningRequest |
| `policy` | PodDisruptionBudget |
| `apiextensions.k8s.io` | CustomResourceDefinition |
| `coordination.k8s.io` | Lease |

---

## Exploring API Groups

```bash
# List all API groups and versions
kubectl api-versions

# List all API resources with their groups
kubectl api-resources

# Filter by group
kubectl api-resources --api-group=apps
kubectl api-resources --api-group=rbac.authorization.k8s.io

# Check a resource's group and kind
kubectl api-resources | grep deployment
```

---

## Direct API Exploration via curl

```bash
# List all groups
curl -k https://localhost:6443/apis

# List resources in a specific group
curl -k https://localhost:6443/apis/apps/v1

# Core group resources
curl -k https://localhost:6443/api/v1
```

---

## API Versions: alpha, beta, stable

| Suffix | Maturity | Example |
|--------|---------|---------|
| `v1alpha1` | Experimental, may change/be removed | `v1alpha1` |
| `v1beta1` | Feature-complete, API may change | `v1beta1` |
| `v1` | Stable, no breaking changes | `v1`, `apps/v1` |

---

## kubectl explain — API Reference

```bash
# Get full schema of a resource
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy

# Show full recursive tree
kubectl explain pod --recursive | head -50
```

---

## API Groups in RBAC

When writing RBAC rules, you must specify the correct `apiGroup`:

```yaml
rules:
- apiGroups: [""]          # core group — pods, services, etc.
  resources: ["pods"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]      # apps group — deployments, etc.
  resources: ["deployments"]
  verbs: ["get", "list"]
- apiGroups: ["batch"]     # batch group — jobs
  resources: ["jobs"]
  verbs: ["create", "delete"]
```
