# Scheduler Profiles — Practical Exercises

These exercises modify the kube-scheduler's static pod manifest. Practice on a throwaway minikube/kind cluster, never production.

## Setup
```bash
kubectl create namespace splab
kubectl config set-context --current --namespace=splab
```

---

## Section A — Inspect the Current Scheduler Config

For minikube:
```bash
minikube ssh
sudo cat /etc/kubernetes/manifests/kube-scheduler.yaml
sudo ls -la /etc/kubernetes/   # look for scheduler-config.yaml or similar
exit
```
By default kubeadm doesn't ship a `--config` flag — the scheduler uses defaults.

---

## Section B — Add a Custom Config With Two Profiles

### B1. Drop a config file on the node
```bash
minikube ssh <<'EOF'
sudo tee /etc/kubernetes/scheduler-config.yaml > /dev/null <<'YAML'
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
- schedulerName: bin-packer
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: MostAllocated
leaderElection:
  leaderElect: false
YAML
exit
EOF
```

### B2. Patch the static pod manifest to use it
```bash
minikube ssh <<'EOF'
sudo sed -i '/    - kube-scheduler/a\    - --config=/etc/kubernetes/scheduler-config.yaml' /etc/kubernetes/manifests/kube-scheduler.yaml

# Add a volumeMount and volume for the config file
sudo python3 - <<'PY'
import yaml, sys
p = '/etc/kubernetes/manifests/kube-scheduler.yaml'
with open(p) as f: m = yaml.safe_load(f)
c = m['spec']['containers'][0]
c.setdefault('volumeMounts', []).append({'name':'sched-cfg','mountPath':'/etc/kubernetes/scheduler-config.yaml','readOnly':True,'subPath':'scheduler-config.yaml'})
m['spec'].setdefault('volumes', []).append({'name':'sched-cfg','hostPath':{'path':'/etc/kubernetes/scheduler-config.yaml','type':'File'}})
with open(p,'w') as f: yaml.safe_dump(m, f)
PY
exit
EOF

# Wait for kubelet to restart kube-scheduler
sleep 15
kubectl get pods -n kube-system -l component=kube-scheduler
```

### B3. Verify the new flag is present
```bash
kubectl logs -n kube-system kube-scheduler-$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') | grep -i profile | head -5
```

---

## Section C — Use Each Profile

### C1. Default profile pod
```bash
kubectl run pod-default --image=nginx
kubectl get pod pod-default -o jsonpath='{.spec.schedulerName}{"\n"}'
# default-scheduler
```

### C2. Bin-packer profile pod
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: pod-binpack }
spec:
  schedulerName: bin-packer
  containers: [{ name: c, image: nginx }]
EOF

kubectl get pod pod-binpack -o jsonpath='{.spec.schedulerName}{"\n"}'
kubectl get events --field-selector source.component=bin-packer --sort-by='.lastTimestamp' | tail -5
```

---

## Section D — Try a "no preemption" Profile

Add a third profile to the config:
```yaml
- schedulerName: no-preempt
  plugins:
    postFilter:
      disabled:
      - name: DefaultPreemption
```
After restarting the scheduler, pods using `schedulerName: no-preempt` will never evict anyone.

---

## Section E — Restore Default Config

When you're done experimenting:
```bash
minikube ssh <<'EOF'
# Remove the --config arg and config file
sudo sed -i '\|--config=/etc/kubernetes/scheduler-config.yaml|d' /etc/kubernetes/manifests/kube-scheduler.yaml
sudo rm -f /etc/kubernetes/scheduler-config.yaml
exit
EOF

sleep 10
kubectl get pods -n kube-system -l component=kube-scheduler
```

---

## Section F — Cleanup
```bash
kubectl delete pod pod-default pod-binpack --ignore-not-found
kubectl config set-context --current --namespace=default
kubectl delete namespace splab
```
