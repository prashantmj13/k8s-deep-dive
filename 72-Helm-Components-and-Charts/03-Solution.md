# 72 — Helm Components and Charts | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 25 minutes

---

## Exercise 1 — Create a new chart

```bash
helm create mychart
find mychart -type f
cat mychart/Chart.yaml
```

---

## Exercise 2 — Render templates locally (no install)

```bash
helm template mychart/ | head -60
helm template mychart/ --set replicaCount=5 | grep replicas
```

---

## Exercise 3 — Lint the chart

```bash
helm lint mychart/

# Introduce an error to see lint catch it
echo "  badIndent: true" >> mychart/values.yaml
helm lint mychart/
git checkout mychart/values.yaml 2>/dev/null || sed -i '$ d' mychart/values.yaml
```

---

## Exercise 4 — Modify values.yaml and re-render

```bash
sed -i 's/replicaCount: 1/replicaCount: 3/' mychart/values.yaml
helm template mychart/ | grep -A1 "kind: Deployment" 
helm template mychart/ | grep replicas
```

---

## Exercise 5 — Add a named template (_helpers.tpl)

```bash
cat mychart/templates/_helpers.tpl | head -20
```

Add a new helper:
```
{{- define "mychart.env" -}}
{{ .Values.environment | default "production" }}
{{- end }}
```

Use it in a template and render:
```bash
echo 'environment: staging' >> mychart/values.yaml
helm template mychart/ --show-only templates/deployment.yaml | head -20
```

---

## Exercise 6 — Package the chart

```bash
helm package mychart/
ls -lh mychart-*.tgz
tar -tzf mychart-*.tgz | head -10
```

---

## Exercise 7 — Install from the packaged tgz

```bash
kubectl create namespace chart-lab
helm install demo mychart-0.1.0.tgz -n chart-lab
helm get manifest demo -n chart-lab | head -30
```

---

## Exercise 8 — Cleanup

```bash
helm uninstall demo -n chart-lab
kubectl delete namespace chart-lab
rm -rf mychart mychart-*.tgz
```

---

## Challenge Questions

1. What is the difference between chart `version` and `appVersion` in Chart.yaml?
2. What does `helm template` do differently from `helm install`?
3. What is a "library" chart type used for?
4. Why does `helm lint` matter before packaging/distributing a chart?
5. What built-in object would you use inside a template to access the release name?

---

# Solutions

**Exercise 4 output:**
```yaml
kind: Deployment
spec:
  replicas: 3
```

**Exercise 6:**
```
mychart-0.1.0.tgz
mychart/Chart.yaml
mychart/values.yaml
mychart/templates/deployment.yaml
...
```

## Challenge Answers

**1. version vs appVersion?**
`version` is the chart's own semantic version — bump it whenever you change templates, defaults, or chart structure. `appVersion` documents what version of the actual application the chart deploys (e.g. nginx 1.25.3) — it's purely informational and doesn't affect chart resolution/installation.

**2. helm template vs helm install?**
`helm template` renders the chart's manifests to stdout locally — no cluster interaction, no release created, no validation against a live API server. `helm install` does all of that AND actually applies the resources, creates a release record, and runs any hooks. `template` is the safe way to preview output before committing.

**3. Library chart purpose?**
A library chart contains only reusable named templates (`_helpers.tpl`-style definitions) with no deployable resources of its own. Other charts declare it as a dependency to share common logic (e.g., standard label sets, common container security context) without duplicating YAML across many charts.

**4. Why lint matters?**
`helm lint` catches structural problems (bad YAML, missing required fields, invalid template syntax) before you package and distribute a chart — catching errors at author-time rather than at a user's `helm install` time, which is much harder to debug remotely.

**5. Accessing release name in a template?**
`{{ .Release.Name }}` — part of the built-in `.Release` object available in every template, giving you the name the user chose at `helm install <name> chart/`.
