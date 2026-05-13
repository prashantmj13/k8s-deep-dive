# 51 — Custom Resource Definitions (CRD)

## What is a CRD?

A **Custom Resource Definition** extends the Kubernetes API with your own resource types. Once a CRD is created, you can use `kubectl` to create, get, list, delete instances of your custom resource — just like Pods or Deployments.

```bash
# After creating a CRD for "AppConfig":
kubectl get appconfigs
kubectl apply -f my-appconfig.yaml
kubectl describe appconfig my-app
```

---

## Use Cases

- Operators (databases, message queues, ML pipelines)
- Application configuration (store app settings as Kubernetes objects)
- Platform abstractions (hide complex configs behind simple APIs)
- GitOps (ArgoCD, Flux store their state as CRDs)

---

## Creating a CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: appconfigs.stable.example.com   # must be <plural>.<group>
spec:
  group: stable.example.com
  versions:
  - name: v1
    served: true      # this version is active
    storage: true     # this version is stored in etcd
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              replicas:
                type: integer
                minimum: 1
                maximum: 10
              image:
                type: string
              environment:
                type: string
                enum: ["dev", "staging", "production"]
  scope: Namespaced   # or Cluster
  names:
    plural: appconfigs
    singular: appconfig
    kind: AppConfig
    shortNames:
    - ac
```

```bash
kubectl apply -f appconfig-crd.yaml
kubectl get crds
kubectl api-resources | grep appconfig
```

---

## Creating a Custom Resource (CR)

Once the CRD exists, create instances:

```yaml
apiVersion: stable.example.com/v1
kind: AppConfig
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 3
  image: nginx:1.25
  environment: production
```

```bash
kubectl apply -f my-appconfig.yaml
kubectl get appconfigs
kubectl get ac   # using shortName
kubectl describe appconfig my-app
```

---

## CRD Schema Validation

The `openAPIV3Schema` field validates CR fields:

```yaml
schema:
  openAPIV3Schema:
    type: object
    required: ["spec"]
    properties:
      spec:
        type: object
        required: ["image"]
        properties:
          image:
            type: string
          replicas:
            type: integer
            minimum: 1
            maximum: 100
```

If you try to create a CR that violates the schema:
```
error: AppConfig.stable.example.com "bad-app" is invalid:
  spec.replicas: Invalid value: 200: spec.replicas in body should be less than or equal to 100
```

---

## CRD Additional Printer Columns

Customize what `kubectl get` shows:

```yaml
additionalPrinterColumns:
- name: Replicas
  type: integer
  jsonPath: .spec.replicas
- name: Environment
  type: string
  jsonPath: .spec.environment
- name: Age
  type: date
  jsonPath: .metadata.creationTimestamp
```

Output:
```
NAME     REPLICAS   ENVIRONMENT   AGE
my-app   3          production    5m
```

---

## CRD Lifecycle Commands

```bash
# List all CRDs
kubectl get crds

# Describe a CRD
kubectl describe crd appconfigs.stable.example.com

# Delete a CRD (also deletes all instances!)
kubectl delete crd appconfigs.stable.example.com

# List all instances of a custom resource
kubectl get appconfigs -A
```

---

## CRD vs ConfigMap

| | ConfigMap | CRD |
|-|-----------|-----|
| RBAC | Shared with all ConfigMaps | Can be granted independently |
| Validation | None | Schema validation |
| Watch/Informers | Generic | Type-specific |
| kubectl get | `kubectl get cm` | `kubectl get appconfig` |
| Best for | Simple key-value config | Structured config, operator state |
