# Static Pods — Solutions

## Section A — Existing
The four control-plane pods are static pods. Their YAMLs live in `/etc/kubernetes/manifests/`. The mirror pod's annotation `kubernetes.io/config.source: file` distinguishes them from API-managed pods (which would say `api`).

## Section B — staticPodPath
```yaml
staticPodPath: /etc/kubernetes/manifests
```
Path is configurable per kubelet. You can change it and restart the kubelet to use a different directory.

## Section C — Your Own Static Pod
Pod name in the API is `myapp-<nodename>` because the kubelet appends the node name to make it globally unique.

## Section D — Delete via kubectl
**Answer:** the pod briefly disappears from the mirror, then reappears within a few seconds. The kubelet sees the source file is still there and re-creates the mirror. **You cannot delete a static pod via the API.** Only removing the file works.

## Section E — Real Delete
Removing the file removes the pod permanently. The kubelet sees the file is gone and stops the container, then deletes the mirror.

## Section F — Edit
Editing the YAML triggers an immediate restart with the new spec. The kubelet treats the file as the source of truth and reconciles to match.

## Cheat Sheet
| Action | How |
|---|---|
| Create | Write YAML to `/etc/kubernetes/manifests/X.yaml` |
| Delete | Remove the file |
| Update | Edit the file |
| List from API | `kubectl get pods -A` (look for `<name>-<nodename>`) |
| Identify | annotation `kubernetes.io/config.source: file` |
| Find path | `grep staticPodPath /var/lib/kubelet/config.yaml` |

**Use cases:** control-plane bootstrap, kubeadm-managed components. **Avoid for application workloads.**
