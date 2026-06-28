# 71 — Helm Intro and Installation | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 20 minutes

---

## Exercise 1 — Install Helm and verify

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## Exercise 2 — Add repositories

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm repo list
```

---

## Exercise 3 — Search for charts

```bash
helm search repo nginx
helm search repo redis
helm search hub wordpress | head -10
```

---

## Exercise 4 — Inspect a chart before installing

```bash
helm show chart bitnami/nginx
helm show values bitnami/nginx | head -40
helm show readme bitnami/nginx | head -20
```

---

## Exercise 5 — Install your first release

```bash
kubectl create namespace helm-lab
helm install my-nginx bitnami/nginx -n helm-lab
helm list -n helm-lab
kubectl get all -n helm-lab
```

---

## Exercise 6 — Inspect release storage

```bash
kubectl get secrets -n helm-lab -l owner=helm
kubectl get secret -n helm-lab <secret-name> -o jsonpath='{.data.release}' | base64 -d | base64 -d | gunzip | head -50
```

---

## Exercise 7 — Cleanup

```bash
helm uninstall my-nginx -n helm-lab
kubectl delete namespace helm-lab
```

---

## Challenge Questions

1. Why doesn't Helm 3 need Tiller, unlike Helm 2?
2. Where is Helm release state stored?
3. What is the difference between `helm search repo` and `helm search hub`?
4. What command lets you preview a chart's default values before installing?
5. What does `helm repo update` actually do?

---

# Solutions

**Exercise 5 output:**
```
NAME        CHART VERSION   APP VERSION   STATUS      REVISION
my-nginx    15.x.x          1.25.x        deployed    1
```

**Exercise 6:** The decoded secret reveals a JSON manifest of the full rendered chart — this is exactly how Helm performs rollbacks: it stores a complete snapshot of every release revision.

## Challenge Answers

**1. Why no Tiller in Helm 3?**
Tiller was a server-side component with cluster-wide privileges, a major security concern (effectively a privilege escalation vector since any Helm client connecting could deploy with Tiller's permissions). Helm 3 removed it — the CLI now talks directly to the Kubernetes API using your own kubeconfig credentials and RBAC permissions, which is far more secure and auditable.

**2. Release state storage?**
As Kubernetes **Secrets** in the release's namespace, labeled `owner=helm`. Each revision gets its own secret, enabling `helm rollback` to retrieve and reapply any previous state.

**3. search repo vs search hub?**
`helm search repo` searches charts in repositories you've added locally with `helm repo add`. `helm search hub` searches Artifact Hub (hub.helm.sh), a centralized catalog across many publishers — useful for discovering new charts you haven't added yet.

**4. Preview default values?**
`helm show values <repo>/<chart>` — prints the chart's default `values.yaml` so you know what's configurable before installing.

**5. What does repo update do?**
Downloads the latest `index.yaml` from each added repository, refreshing your local cache of available chart versions. Without running it periodically, `helm search` and `helm install` may show/use stale chart versions.
