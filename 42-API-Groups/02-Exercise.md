# 42 — API Groups | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 15 minutes

---

## Exercise 1 — List all API versions

```bash
kubectl api-versions | sort
```

How many groups are there?

---

## Exercise 2 — List all API resources

```bash
kubectl api-resources --sort-by=name | head -30
kubectl api-resources | wc -l
```

---

## Exercise 3 — Filter resources by group

```bash
kubectl api-resources --api-group=apps
kubectl api-resources --api-group=rbac.authorization.k8s.io
kubectl api-resources --api-group=networking.k8s.io
kubectl api-resources --api-group=""   # core group
```

---

## Exercise 4 — Find which group a resource belongs to

```bash
kubectl api-resources | grep -E "^NAME|ingress|networkpolic|cronjob|hpa"
```

---

## Exercise 5 — Explore schema with kubectl explain

```bash
kubectl explain deployment
kubectl explain deployment.spec.strategy
kubectl explain pod.spec.containers.resources
kubectl explain horizontalpodautoscaler.spec
```

---

## Exercise 6 — Identify correct apiGroup for RBAC

For each resource, find the correct apiGroup to use in a Role:

```bash
kubectl api-resources | grep -E "jobs|cronjobs|ingresses|storageclasses"
```

---

## Challenge Questions

1. What is the `apiGroup` value for pods and services in an RBAC rule?
2. What is the difference between `v1alpha1`, `v1beta1`, and `v1`?
3. How do you find what API group a CRD belongs to?
4. What command shows the full YAML schema for a resource field?
5. What URL path serves core API resources vs named group resources?
