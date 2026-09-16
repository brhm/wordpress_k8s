# WordPress on Kubernetes (minikube)

Deploy WordPress on a local single-node Kubernetes cluster (minikube), running **2 WordPress replicas** behind one Service, backed by a single MySQL database. Uploads and `/var/www/html` are shared across replicas through an **NFS ReadWriteMany (RWX)** volume.

## Architecture

```
User → NodePort Service → WordPress (2 replicas) → mysql Service → MySQL → PVC (RWO)
                              │
                              └── shared /var/www/html → RWX PVC (NFS)
```

- WordPress runs as a Deployment with 2 replicas; the Service load-balances between them.
- All replicas share one RWX volume, so uploaded media and files stay consistent.
- MySQL runs as a single pod with its own persistent volume (RWO).
- Fixed WordPress auth keys/salts keep sessions stable across replicas.

## Requirements

- Docker Desktop (as the minikube driver)
- minikube
- kubectl
- Helm 3

## Setup

### 1. Start the cluster

```bash
minikube start
```

### 2. Install an NFS provisioner (for RWX storage)

minikube's default StorageClass only supports RWO. Add an NFS provisioner to get a `ReadWriteMany` StorageClass named `nfs`:

```bash
helm repo add nfs-ganesha-server-and-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-ganesha-server-and-external-provisioner/
helm repo update

helm install nfs-server \
  nfs-ganesha-server-and-external-provisioner/nfs-server-provisioner \
  --set storageClass.name=nfs \
  --set persistence.enabled=true \
  --set persistence.size=5Gi
```

Verify:

```bash
kubectl get storageclass   # expect a "nfs" class alongside "standard"
```

### 3. Deploy WordPress

Apply the manifests in order:

```bash
kubectl apply -f 00-namespace-secret.yaml
kubectl apply -f 01-pvc.yaml
kubectl apply -f 02-mysql.yaml
kubectl apply -f 04-wordpress-pvc-rwx.yaml
kubectl apply -f 03-wordpress.yaml
```

Wait for all pods to be ready:

```bash
kubectl get pods -n wordpress -w
```

### 4. Access

```bash
minikube service wordpress -n wordpress --url
```

Open the printed URL in your browser.

## Manifests

| File | Purpose |
|------|---------|
| `00-namespace-secret.yaml` | `wordpress` namespace + MySQL credentials (Secret) |
| `01-pvc.yaml` | RWO PVC for MySQL data |
| `02-mysql.yaml` | MySQL Deployment + headless Service |
| `03-wordpress.yaml` | WordPress Deployment (2 replicas) + NodePort Service |
| `04-wordpress-pvc-rwx.yaml` | RWX PVC (`storageClassName: nfs`) for shared `/var/www/html` |

## Notes

- **Why RWX?** With multiple replicas and no shared storage, logins drop and image uploads fail, because each request may hit a different pod that doesn't have the other's files. The RWX volume gives all replicas one shared filesystem.
- **Why fixed auth keys?** WordPress generates random session keys per pod by default, so replicas don't recognize each other's cookies. Pinning `WORDPRESS_*_KEY` / `WORDPRESS_*_SALT` env vars keeps sessions stable.
- Replace all example passwords and auth keys/salts with your own values before any real use. Do not commit real secrets.

## Cleanup

```bash
kubectl delete namespace wordpress
helm uninstall nfs-server
minikube stop
```
