# Kubernetes NGINX Deployment Template

Sample Kubernetes manifests that deploy NGINX with a ConfigMap, Secrets, a
PersistentVolumeClaim, two Services (ClusterIP + NodePort), and an Ingress.

| File | Resource | Purpose |
|---|---|---|
| `configmap.yaml` | ConfigMap `nginx-configmap` | Welcome message (env var) + custom `default.conf` with a `/healthz` endpoint |
| `secret.yaml` | Secret `nginx-basic-auth` | Username/password injected as env vars |
| | Secret `nginx-tls` | TLS certificate used by the Ingress and mounted into the pod |
| `pvc.yaml` | PersistentVolumeClaim `nginx-pvc` | 1Gi volume for persistent web content |
| `deployment.yaml` | Deployment `nginx-deployment` | 3 replicas of `nginx:latest` with probes, resource limits, and all mounts |
| `service-clusterip.yaml` | Service `nginx-clusterip` | Cluster-internal access on port 80 |
| `service-nodeport.yaml` | Service `nginx-nodeport` | External access on `<nodeIP>:30080` |
| `ingress.yaml` | Ingress `nginx-ingress` | TLS-terminated routing for `nginx.example.com` |

## Prerequisites

- A running Kubernetes cluster (minikube, kind, Docker Desktop, EKS, GKE, ...)
- `kubectl` installed and configured (`kubectl cluster-info` works)
- An Ingress controller (e.g. [ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/)) — required only for `ingress.yaml`

## Before You Deploy

1. **TLS cert** — `secret.yaml` contains placeholder cert/key. Generate a real self-signed one:

   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
     -keyout tls.key -out tls.crt -subj "/CN=nginx.example.com"

   kubectl create secret tls nginx-tls --cert=tls.crt --key=tls.key \
     --dry-run=client -o yaml | kubectl apply -f -
   ```

   Or paste the contents of `tls.crt` / `tls.key` into `secret.yaml`.

2. **Credentials** — replace the `username` / `password` placeholders in `secret.yaml`.

3. **Storage** — if your cluster has no default StorageClass, uncomment and set
   `storageClassName` in `pvc.yaml`.

4. **Hostname** — change `nginx.example.com` in `ingress.yaml` to your own domain.

## Deploy

Apply everything in one shot (order doesn't matter — Kubernetes resolves
dependencies):

```bash
kubectl apply -f k8sTemplate/
```

Or file by file:

```bash
kubectl apply -f k8sTemplate/configmap.yaml \
              -f k8sTemplate/secret.yaml \
              -f k8sTemplate/pvc.yaml \
              -f k8sTemplate/deployment.yaml \
              -f k8sTemplate/service-clusterip.yaml \
              -f k8sTemplate/service-nodeport.yaml \
              -f k8sTemplate/ingress.yaml
```

## Verify

```bash
kubectl get all -l app=nginx
kubectl get pods -l app=nginx -w          # wait until READY 3/3
kubectl get pvc                          # should be Bound
kubectl describe ingress nginx-ingress   # shows address and TLS
kubectl logs deployment/nginx-deployment
```

## Access the Application

- **NodePort** (no Ingress needed):

  ```bash
  # minikube: minikube service nginx-nodeport --url
  # otherwise: http://<any-node-ip>:30080
  ```

- **Ingress** (requires an Ingress controller). Map the hostname, then browse
  with HTTPS:

  ```bash
  # local clusters (kind/minikube/Docker Desktop)
  echo "127.0.0.1 nginx.example.com" | sudo tee -a /etc/hosts
  # cloud: point DNS at the ingress address from `kubectl get ingress`

  curl -k https://nginx.example.com/healthz   # -> ok
  ```

## Populate the Persistent Volume

The PVC mounts at `/usr/share/nginx/html` and starts empty. Copy content into
a pod to make it persist across restarts:

```bash
kubectl cp ./index.html nginx-deployment-<pod-id>:/usr/share/nginx/html/
```

## Cleanup

```bash
kubectl delete -f k8sTemplate/
```

> **Note:** the PVC is `ReadWriteOnce`. On a multi-node cluster, only the pod
> scheduled on the volume's node can mount it — use `ReadWriteMany` (with a
> provider that supports it) or set `replicas: 1` for a portable demo.

> **Note:** `nginx:latest` is fine for demos. Pin a version (e.g. `nginx:1.27.0`)
> in `deployment.yaml` for production, and `imagePullPolicy: IfNotPresent`.
