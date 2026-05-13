# 27 — HPA | Solutions & Expected Outputs

---

## Exercise 3 — HPA Created

```bash
kubectl get hpa
```
```
NAME         REFERENCE               TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   <unknown>/50%   1         5         1          10s
```
After ~1 min:
```
php-apache   Deployment/php-apache   2%/50%          1         5         1          90s
```

---

## Exercise 5 — Under Load

```bash
kubectl get hpa php-apache-hpa -w
```
```
NAME              REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS
php-apache-hpa    Deployment/php-apache   2%/50%     1         5         1
php-apache-hpa    Deployment/php-apache   68%/50%    1         5         1
php-apache-hpa    Deployment/php-apache   68%/50%    1         5         2
php-apache-hpa    Deployment/php-apache   85%/50%    1         5         3
php-apache-hpa    Deployment/php-apache   62%/50%    1         5         4
```

```bash
kubectl get pods -w
```
```
NAME                          READY   STATUS    REPLICAS
php-apache-xxx                1/1     Running
php-apache-yyy                0/1     Pending    ← HPA adding pods
php-apache-yyy                1/1     Running
```

---

## Exercise 6 — Scale Down

After deleting load-gen, CPU drops. HPA waits 5 minutes (stabilizationWindowSeconds=300) then scales down one pod at a time.

```
php-apache-hpa   Deployment/php-apache   3%/50%   1   5   4
php-apache-hpa   Deployment/php-apache   1%/50%   1   5   3
php-apache-hpa   Deployment/php-apache   0%/50%   1   5   2
php-apache-hpa   Deployment/php-apache   0%/50%   1   5   1
```

---

## Exercise 7 — Describe with Behaviour

```bash
kubectl describe hpa php-apache-hpa
```
```
Name:               php-apache-hpa
Namespace:          hpa-lab
Reference:          Deployment/php-apache
Metrics:            ( current / target )
  resource cpu:     2% (4m) / 50%
Min replicas:       1
Max replicas:       5
Deployment pods:    1 current / 1 desired
Conditions:
  Type            Status  Reason
  AbleToScale     True    ScaleDownStabilized
  ScalingActive   True    ValidMetricFound
  ScalingLimited  False   DesiredWithinRange
```

---

## Challenge Answers

**1. Manual scale vs HPA?**
HPA will override the manual scale on its next evaluation cycle (~15s). The HPA controller continuously reconciles the desired replica count based on metrics. If you want to pin replicas, you must delete the HPA first.

**2. Why resource requests are required?**
HPA calculates utilization as `current usage / request`. Without a request, there is no baseline — Kubernetes cannot compute a percentage. Pods without requests show `<unknown>` in HPA targets and HPA cannot act.

**3. Default scale-down stabilization window?**
300 seconds (5 minutes). This prevents "thrashing" — rapidly scaling down and then back up if load is temporarily low. The HPA keeps track of the highest desired replica count over the stabilization window and uses that.

**4. Utilization vs AverageValue?**
- `Utilization`: percentage of the resource request. `50` means scale when average pod usage exceeds 50% of its CPU request.
- `AverageValue`: absolute value per pod. `200Mi` means scale when average pod memory usage exceeds 200Mi regardless of requests.

**5. Can HPA scale a StatefulSet?**
Yes. HPA `scaleTargetRef` supports `Deployment`, `ReplicaSet`, `StatefulSet`, and any resource that implements the scale subresource.
