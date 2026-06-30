# Airflow

Apache Airflow deployment for the anime-lakehouse project. Used as a pure orchestrator — all data processing runs in isolated Kubernetes pods via `KubernetesPodOperator`.

## Architecture

- **Executor**: LocalExecutor (no Redis, no Celery workers)
- **DAGs**: synced from the `dags/` directory of this repository via git-sync
- **Database**: PostgreSQL bundled via Helm chart
- **Secrets**: managed via Sealed Secrets

## Access the UI locally

```bash
kubectl port-forward svc/airflow-webserver -n airflow 8080:8080
```

Then open: http://localhost:8080

Initial credentials: `admin` / `admin` — change immediately after first login.

## Secret setup

Two sealed secrets are required before syncing:

### 1. Airflow credentials (fernet key + webserver secret)

Generate the values:

```bash
# Fernet key
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"

# Webserver secret key
python -c "import secrets; print(secrets.token_hex(32))"
```

Fill in `secret-template-credentials.yaml` and seal it:

```bash
kubeseal --controller-name=sealed-secrets-controller \
         --controller-namespace=kube-system \
         --format yaml \
         -f secret-template-credentials.yaml > airflow-sealed-secret.yaml
```

### 2. Git credentials (for DAG git-sync)

Create a GitHub personal access token with read access to the repository. Fill in `secret-template-git.yaml` and seal it:

```bash
kubeseal --controller-name=sealed-secrets-controller \
         --controller-namespace=kube-system \
         --format yaml \
         -f secret-template-git.yaml > airflow-git-sealed-secret.yaml
```

Add both generated files to `kustomization.yaml` resources after sealing.

## Helm chart

The deployment is managed by Argo CD using the official Apache Airflow Helm chart.

- Chart: `apache-airflow/airflow`
- Version: see `infra/kubernetes/base/apps/airflow.yaml`
- Custom values: `values.yaml`

## DAG development

DAGs live in `dags/` at the root of this repository. git-sync polls the `main` branch every 60 seconds and makes them available to the scheduler automatically.
