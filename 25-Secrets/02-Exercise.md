# 25 — Secrets | Exercises

> **Cluster:** Any kubectl-accessible cluster (minikube / kind / GKE)
> **Estimated time:** 20–25 minutes
> **Namespace:** `secrets-lab` (created in Exercise 1)

---

## Exercise 1 — Create the namespace

```bash
kubectl create namespace secrets-lab
kubectl config set-context --current --namespace=secrets-lab
```

---

## Exercise 2 — Create a Secret imperatively (generic)

Create a secret called `db-secret` with two key-value pairs:
- `DB_USER=admin`
- `DB_PASSWORD=supersecret123`

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=supersecret123
```

Verify it was created:

```bash
kubectl get secret db-secret
kubectl describe secret db-secret
```

> **Question:** Why does `describe` not show the actual values?

---

## Exercise 3 — Decode a Secret manually

The values are base64-encoded. Decode them:

```bash
kubectl get secret db-secret -o jsonpath='{.data.DB_USER}' | base64 --decode
kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode
```

---

## Exercise 4 — Create a Secret from a file

First create a file with a "private key":

```bash
echo -n "my-super-private-key-content" > /tmp/api.key
```

Now create a secret from it:

```bash
kubectl create secret generic api-key-secret \
  --from-file=api.key=/tmp/api.key
```

Verify:

```bash
kubectl get secret api-key-secret -o yaml
```

---

## Exercise 5 — Create a Secret declaratively (YAML)

Create a file `tls-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: secrets-lab
type: Opaque
data:
  TLS_CERT: dGxzLWNlcnQtY29udGVudA==   # base64 of "tls-cert-content"
  TLS_KEY: dGxzLWtleS1jb250ZW50         # base64 of "tls-key-content"
```

Apply it:

```bash
kubectl apply -f tls-secret.yaml
kubectl get secrets
```

---

## Exercise 6 — Inject Secret as Environment Variables

Create `pod-env-secret.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-env-secret
  namespace: secrets-lab
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo DB_USER=$DB_USER && echo DB_PASSWORD=$DB_PASSWORD && sleep 3600"]
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: DB_USER
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: DB_PASSWORD
  restartPolicy: Never
```

```bash
kubectl apply -f pod-env-secret.yaml
kubectl logs pod-env-secret
```

> **Expected:** You should see the actual decoded values printed.

---

## Exercise 7 — Inject ALL keys from a Secret using envFrom

Create `pod-envfrom-secret.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-envfrom-secret
  namespace: secrets-lab
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "env | grep -E 'DB_USER|DB_PASSWORD' && sleep 3600"]
    envFrom:
    - secretRef:
        name: db-secret
  restartPolicy: Never
```

```bash
kubectl apply -f pod-envfrom-secret.yaml
kubectl logs pod-envfrom-secret
```

---

## Exercise 8 — Mount a Secret as a Volume

Create `pod-volume-secret.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-volume-secret
  namespace: secrets-lab
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "ls /etc/db-creds && cat /etc/db-creds/DB_USER && sleep 3600"]
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/db-creds
      readOnly: true
  volumes:
  - name: secret-vol
    secret:
      secretName: db-secret
  restartPolicy: Never
```

```bash
kubectl apply -f pod-volume-secret.yaml
kubectl logs pod-volume-secret
```

> **Observe:** Each key becomes a **file** inside `/etc/db-creds/`. The filename = key, file content = decoded value.

---

## Exercise 9 — Update a Secret and observe live reload

Update `DB_PASSWORD` to a new value:

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=newpassword456 \
  --dry-run=client -o yaml | kubectl apply -f -
```

Check if the volume-mounted pod sees the new value (may take ~60s to propagate):

```bash
kubectl exec pod-volume-secret -- cat /etc/db-creds/DB_PASSWORD
```

> **Note:** Volume-mounted secrets update automatically. Env var secrets do **not** — the pod must be restarted.

---

## Exercise 10 — Create a docker-registry Secret

Simulate creating a registry pull secret:

```bash
kubectl create secret docker-registry my-registry-secret \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myuser@example.com
```

Inspect its type:

```bash
kubectl get secret my-registry-secret -o jsonpath='{.type}'
```

> **Expected output:** `kubernetes.io/dockerconfigjson`

---

## Exercise 11 — Cleanup

```bash
kubectl delete namespace secrets-lab
kubectl config set-context --current --namespace=default
```

---

## Challenge Questions

1. What is the difference between `Opaque` and `kubernetes.io/dockerconfigjson` secret types?
2. Why are Secrets only base64-encoded and not truly encrypted by default?
3. What Kubernetes feature enables encryption of Secrets at rest in etcd?
4. When would you use a volume mount vs env var for a Secret?
5. Does updating a Secret immediately update env vars in a running pod?
