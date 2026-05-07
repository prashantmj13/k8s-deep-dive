# Rolling Update and Rollback — Exercises

## Setup
```bash
kubectl create namespace rulab
kubectl config set-context --current --namespace=rulab
```

---

## Section A — Five Rollouts in a Row

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl annotate deployment web kubernetes.io/change-cause="initial 1.25" --overwrite

for tag in 1.26 1.27 1.27-alpine 1.27 1.28; do
  kubectl set image deployment/web nginx=nginx:$tag
  kubectl annotate deployment web kubernetes.io/change-cause="upgrade $tag" --overwrite
  kubectl rollout status deployment/web
done

kubectl rollout history deployment/web
```

---

## Section B — Roll Back to Earlier Revision

```bash
kubectl rollout history deployment/web --revision=2
kubectl rollout undo deployment/web --to-revision=2
kubectl rollout status deployment/web
kubectl get pods -l app=web -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}'
```

---

## Section C — Pause / Resume

```bash
kubectl rollout pause deployment/web
kubectl set image deployment/web nginx=nginx:1.28
kubectl set env  deployment/web LOG_LEVEL=info
kubectl set resources deployment/web -c nginx --requests=cpu=100m,memory=128Mi
kubectl get pods -l app=web -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}'
# Still showing OLD image — paused

kubectl rollout resume deployment/web
kubectl rollout status deployment/web
kubectl get pods -l app=web -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}'
# All three changes shipped in one rollout
```

---

## Section D — Stuck Rollout + Recovery

### D1. Trigger a bad image
```bash
kubectl set image deployment/web nginx=nginx:does-not-exist-1.99
kubectl get pods -l app=web -o wide
sleep 10
kubectl describe deployment web | grep -A3 Conditions
```

### D2. Tighten the deadline
```bash
kubectl patch deployment web -p '{"spec":{"progressDeadlineSeconds":60}}'
sleep 65
kubectl describe deployment web | grep -A3 Conditions
# Progressing=False, Reason=ProgressDeadlineExceeded
```

### D3. Recover via undo
```bash
kubectl rollout undo deployment/web
kubectl rollout status deployment/web
```

---

## Section E — Restart Without Image Change

After a Secret/ConfigMap change, you want to bounce all pods without modifying the image:
```bash
kubectl rollout restart deployment/web
kubectl rollout status deployment/web
kubectl get deployment web -o jsonpath='{.spec.template.metadata.annotations}'
# kubectl.kubernetes.io/restartedAt was added
```

---

## Section F — revisionHistoryLimit

```bash
kubectl patch deployment web -p '{"spec":{"revisionHistoryLimit":2}}'

# Push a few rollouts
for tag in 1.26 1.27 1.28; do
  kubectl set image deployment/web nginx=nginx:$tag
  kubectl rollout status deployment/web
done

kubectl get rs
# Only 2 inactive RSs plus the active one
```

---

## Section G — Cleanup
```bash
kubectl delete deployment web
kubectl config set-context --current --namespace=default
kubectl delete namespace rulab
```
