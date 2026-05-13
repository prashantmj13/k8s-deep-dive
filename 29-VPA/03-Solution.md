# 29 — VPA | Solutions & Expected Outputs

---

## Exercise 1 — VPA installed

```bash
kubectl get pods -n kube-system | grep vpa
```
```
vpa-admission-controller-xxx   1/1   Running   0   2m
vpa-recommender-xxx            1/1   Running   0   2m
vpa-updater-xxx                1/1   Running   0   2m
```

---

## Exercise 3 — VPA created

```bash
kubectl get vpa
```
```
NAME          MODE   CPU   MEM    PROVIDED   AGE
hamster-vpa   Off    0     0      False      30s
```

`PROVIDED: False` means recommendations not yet available.

---

## Exercise 4 — Recommendations (after 5–10 min)

```bash
kubectl describe vpa hamster-vpa
```
```
Status:
  Recommendation:
    Container Recommendations:
      Container Name:  hamster
      Lower Bound:
        Cpu:     100m
        Memory:  50Mi
      Target:
        Cpu:     587m
        Memory:  262144k
      Uncapped Target:
        Cpu:     587m
        Memory:  262144k
      Upper Bound:
        Cpu:     1
        Memory:  500Mi
```

VPA recommends ~587m CPU — much higher than the 100m request set initially, because the app is doing CPU work in a tight loop.

---

## Exercise 5 — Auto mode pod eviction

```bash
kubectl get pods -w
```
```
NAME                       READY   STATUS    RESTARTS   AGE
hamster-xxx-aaa            1/1     Running   0          8m
hamster-xxx-aaa            1/1     Terminating   0      8m    ← VPA evicts
hamster-xxx-bbb            0/1     Pending       0      0s    ← new pod
hamster-xxx-bbb            1/1     Running       0      5s
```

```bash
kubectl describe pod hamster-xxx-bbb | grep -A6 Requests
```
```
    Requests:
      cpu:     587m      ← set by VPA (was 100m)
      memory:  262144k   ← set by VPA (was 50Mi)
```

---

## Challenge Answers

**1. Auto vs Initial mode?**
- `Auto`: VPA continuously monitors and evicts/recreates pods when their resources drift significantly from recommendations.
- `Initial`: VPA only sets recommended resources when a pod is first created (via admission controller). It will NOT evict running pods. Good for workloads that can't tolerate unexpected evictions.

**2. VPA + HPA on CPU — conflict?**
HPA scales replicas up/down based on current CPU usage. VPA adjusts CPU requests. They fight each other: VPA increases requests → HPA sees utilization drop → HPA scales down → fewer pods → VPA sees high usage again → cycle repeats. Safe combo: HPA on custom/external metrics + VPA on CPU/memory, or VPA memory only + HPA CPU.

**3. Which component evicts pods?**
The **VPA Updater** evicts pods that need resource adjustments. The **VPA Admission Controller** then sets the correct resource values on the newly scheduled replacement pods.

**4. How long for reliable recommendations?**
Generally at least **24 hours** of metric history is recommended for stable results. In tests/labs you may see initial recommendations in 5–10 minutes, but they are based on limited data and may not reflect real workload patterns.

**5. updateMode: Off — does VPA change anything?**
No. In `Off` mode, VPA only **reads** metrics and generates recommendations stored in the VPA object's status. It does NOT evict pods, does NOT modify any pod's resources. It is purely advisory. Use it to get recommendations before committing to automatic changes.
