# 42 — API Groups | Solutions

---

## Exercise 3 — Resources by group

```bash
kubectl api-resources --api-group=apps
```
```
NAME                  SHORTNAMES   APIVERSION   NAMESPACED   KIND
controllerrevisions                apps/v1      true         ControllerRevision
daemonsets            ds           apps/v1      true         DaemonSet
deployments           deploy       apps/v1      true         Deployment
replicasets           rs           apps/v1      true         ReplicaSet
statefulsets          sts          apps/v1      true         StatefulSet
```

---

## Exercise 4 — Resource groups

```
NAME                        APIVERSION                    NAMESPACED   KIND
horizontalpodautoscalers    autoscaling/v2                true         HorizontalPodAutoscaler
cronjobs                    batch/v1                      true         CronJob
ingresses                   networking.k8s.io/v1          true         Ingress
networkpolicies             networking.k8s.io/v1          true         NetworkPolicy
```

---

## Challenge Answers

**1. apiGroup for pods/services in RBAC?**
Use `""` (empty string) — the core API group has no name. Any resource with `apiVersion: v1` belongs to the core group.

**2. v1alpha1 vs v1beta1 vs v1?**
- `v1alpha1`: Early experimental. May be removed without notice. Don't use in production.
- `v1beta1`: Feature-complete but API details may still change. Test in staging.
- `v1`: Stable and supported. Backward-compatible within major version. Use in production.

**3. Find CRD's API group?**
```bash
kubectl api-resources | grep <crd-name>
# or
kubectl get crd <crd-name> -o jsonpath='{.spec.group}'
```

**4. Full YAML schema for a field?**
```bash
kubectl explain <resource>.<field>.<subfield>
# e.g.
kubectl explain pod.spec.containers.livenessProbe
```

**5. URL paths?**
- Core API resources: `https://<apiserver>/api/v1`
- Named group resources: `https://<apiserver>/apis/<group>/<version>`
  e.g. `https://<apiserver>/apis/apps/v1`
