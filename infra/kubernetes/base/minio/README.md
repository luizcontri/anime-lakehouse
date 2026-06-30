# MinIO

Object storage for the anime-lakehouse data lake. Provides an S3-compatible API used by all pipeline stages.

## Architecture

- Single-node standalone deployment (suitable for local development).
- Data is persisted on `desktop-worker` at `/mnt/minio-data` via a `local` PersistentVolume.
- Credentials are managed via Sealed Secrets.
- Exposed internally via ClusterIP on port `9000` (API) and `9001` (console).

## Endpoints (internal)

| Endpoint | Purpose |
|---|---|
| `http://minio.minio.svc.cluster.local:9000` | S3 API (used by Spark, Airflow) |
| `http://minio.minio.svc.cluster.local:9001` | Web console |

## Access the console locally

```bash
kubectl port-forward svc/minio -n minio 9001:9001
```

Then open: http://localhost:9001

## Secret setup

Before applying this directory, generate the sealed secret from the template:

```bash
# Fill in real values first (do not commit this file)
cp secret-template.yaml minio-secret.yaml
# Edit minio-secret.yaml with real credentials

kubeseal --controller-name=sealed-secrets-controller \
         --controller-namespace=kube-system \
         --format yaml \
         -f minio-secret.yaml > minio-sealed-secret.yaml

# Add minio-sealed-secret.yaml to kustomization.yaml resources
# Commit minio-sealed-secret.yaml (it is safe to commit)
# Delete minio-secret.yaml (never commit it)
```

## Storage

Data is stored on `desktop-worker` at `/mnt/minio-data`. The node is labeled `role=storage`.

To label the node on a fresh cluster:

```bash
kubectl label node desktop-worker role=storage
```

## Note on migration to S3

The S3 endpoint in all pipeline configurations uses the `AWS_ENDPOINT_URL` environment variable. To migrate to real S3, update the endpoint and credentials in the sealed secret without changing any pipeline code.
