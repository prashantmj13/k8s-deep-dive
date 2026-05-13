# 51 — CRD | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 20 minutes

---

## Exercise 1 — Create a CRD

```yaml
# appconfig-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: appconfigs.stable.example.com
spec:
  group: stable.example.com
  versions:
  - name: v1
    served: true
    storage: true
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
    additionalPrinterColumns:
    - name: Replicas
      type: integer
      jsonPath: .spec.replicas
    - name: Environment
      type: string
      jsonPath: .spec.environment
  scope: Namespaced
  names:
    plural: appconfigs
    singular: appconfig
    kind: AppConfig
    shortNames: ["ac"]
```

```bash
kubectl apply -f appconfig-crd.yaml
kubectl get crds | grep appconfig
kubectl api-resources | grep appconfig
```

---

## Exercise 2 — Create a Custom Resource

```yaml
# my-appconfig.yaml
apiVersion: stable.example.com/v1
kind: AppConfig
metadata:
  name: my-app
spec:
  replicas: 3
  image: nginx:1.25
  environment: production
```

```bash
kubectl apply -f my-appconfig.yaml
kubectl get appconfigs
kubectl get ac
kubectl describe appconfig my-app
```

---

## Exercise 3 — Test schema validation

```yaml
# bad-appconfig.yaml
apiVersion: stable.example.com/v1
kind: AppConfig
metadata:
  name: bad-app
spec:
  replicas: 50    # exceeds maximum of 10
  environment: unknown   # not in enum
```

```bash
kubectl apply -f bad-appconfig.yaml
# Should fail with validation error
```

---

## Exercise 4 — Use RBAC on a CRD

```bash
kubectl create role appconfig-reader \
  --verb=get,list,watch \
  --resource=appconfigs.stable.example.com

kubectl describe role appconfig-reader
```

---

## Exercise 5 — Delete the CRD

```bash
kubectl delete appconfig my-app
kubectl delete crd appconfigs.stable.example.com
kubectl get appconfigs   # should error — CRD gone
```

---

## Challenge Questions

1. What is the naming convention for a CRD's `metadata.name`?
2. What does `scope: Namespaced` vs `scope: Cluster` mean?
3. What happens to custom resource instances when you delete the CRD?
4. What is the difference between `served: true` and `storage: true`?
5. How do you add tab-completion shortNames for a CRD?
