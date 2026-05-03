# Static Pods — Practical Exercises

These exercises require shell access to a node (minikube ssh, kind exec, or a real SSH session).

---

## Section A — Inspect Existing Static Pods

### A1. Find them on the node
For minikube:
```bash
minikube ssh
sudo ls -la /etc/kubernetes/manifests/
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | head -30
exit
```
For kind:
```bash
docker exec -it <cluster-name>-control-plane bash
ls -la /etc/kubernetes/manifests/
cat /etc/kubernetes/manifests/etcd.yaml | head -30
exit
```

### A2. See the mirror pods in the API
```bash
kubectl get pods -n kube-system | grep -E 'apiserver|etcd|controller-manager|scheduler'
kubectl get pod -n kube-system kube-apiserver-<NODE> -o yaml | grep -A5 annotations
```
Look for `kubernetes.io/config.source: file` — proof it's a static pod.

---

## Section B — Find the Kubelet's Manifest Path

```bash
minikube ssh
sudo grep staticPodPath /var/lib/kubelet/config.yaml
exit
```
Default: `/etc/kubernetes/manifests`.

---

## Section C — Create Your Own Static Pod

### C1. Drop a YAML file
```bash
minikube ssh <<'EOF'
sudo tee /etc/kubernetes/manifests/myapp.yaml > /dev/null <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: default
spec:
  containers:
  - name: c
    image: nginx
YAML
exit
EOF
```

### C2. Wait, then look from outside
```bash
sleep 5
kubectl get pods -A | grep myapp
```
Expected: a pod named `myapp-<nodename>` in `default` namespace.

### C3. Inspect the mirror
```bash
kubectl get pod -n default myapp-$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') -o yaml | grep -A6 annotations
```
You'll see `kubernetes.io/config.source: file`.

---

## Section D — Try to Delete via kubectl

```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod myapp-$NODE
kubectl get pods | grep myapp
```
**Question:** is the pod still there?

---

## Section E — The Real Way to Delete

```bash
minikube ssh
sudo rm /etc/kubernetes/manifests/myapp.yaml
exit

sleep 5
kubectl get pods | grep myapp
```
Now it's gone for good — the file was the source of truth.

---

## Section F — Edit a Static Pod Spec

### F1. Modify the YAML
```bash
minikube ssh <<'EOF'
sudo tee /etc/kubernetes/manifests/myapp.yaml > /dev/null <<'YAML'
apiVersion: v1
kind: Pod
metadata: { name: myapp }
spec:
  containers:
  - name: c
    image: nginx:1.27
YAML
exit
EOF

sleep 10
kubectl get pod myapp-$NODE -o jsonpath='{.spec.containers[0].image}'
```

### F2. The kubelet picks up the change automatically and restarts the pod with the new image.

---

## Section G — Cleanup

```bash
minikube ssh
sudo rm -f /etc/kubernetes/manifests/myapp.yaml
exit
sleep 5
kubectl get pods | grep myapp
```
