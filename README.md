# 🎌 anime-lakehouse

Anime Lakehouse is an open-source data platform for collecting, consolidating and serving anime metadata and user interaction data at scale. The project is designed for a fully local Kubernetes deployment on the user machine, without relying on cloud services at the moment, while remaining compatible with a future migration to cloud environments. It follows a medallion architecture (bronze → silver → gold) and is intended to run with MinIO, Spark Operator, Delta Lake, Apache Airflow and supporting services.

## 📁 Project structure

```text
anime-lakehouse/
├── dags/
├── infra/
│   └── kubernetes/
│       ├── base/
│       ├── overlays/
│       └── charts/
├── scripts/
├── src/
│   ├── pipelines/
│   │   ├── bronze/
│   │   ├── silver/
│   │   └── gold/
│   └── shared/
└── tests/
```

This layout is intended to host Kubernetes manifests, Airflow DAGs, Spark pipeline code and operational utilities in a consistent and scalable way.

---

## 🚀 Getting started from zero

Before installing anything, you need a Kubernetes cluster running locally.

### ✅ Prerequisites

Make sure you have the following available on your machine:

- ☸️ a working Kubernetes cluster
- 🔧 kubectl configured to access that cluster
- 🐳 Docker or an equivalent container runtime if you want to run local Kubernetes tools through containers
- 📦 Git
- 💻 a shell available in your operating system (for example Bash, Zsh, PowerShell or the terminal provided by your OS)

### 1. Start a local Kubernetes cluster

You can use a local Kubernetes solution such as Docker Desktop with Kubernetes enabled, kind, k3d, minikube or any other local runtime that exposes a working Kubernetes API.

Verify that the cluster is available:

```bash
kubectl config get-contexts
kubectl cluster-info
```

You should see a working Kubernetes context and a healthy control plane.

### 2. Install kubectl locally (if needed)

Install kubectl using the official Kubernetes installation guide for your operating system or your package manager. For example:

- Ubuntu/Debian: `apt install kubectl`
- Fedora/RHEL: `dnf install kubectl`
- macOS with Homebrew: `brew install kubectl`

---

## 📋 Recommended installation order

For this project, the goal is to keep the whole stack declarative.

That means:

- 📂 the Kubernetes manifests live in this repository;
- 🔐 the secrets are managed as SealedSecret manifests;
- 🔄 Argo CD applies everything to the cluster from Git.

The practical order is:

1. Install Sealed Secrets first
2. Install Argo CD second
3. Use Argo CD to apply the rest of the stack from this repository

> **Why this order?**
> Sealed Secrets is the controller that turns SealedSecret objects into real Kubernetes Secrets.
> Argo CD can be installed first, but the declarative secret flow becomes usable only after Sealed Secrets is ready.
> Once both are available, the cluster can be managed entirely from manifests in Git.

---

## 🔐 Step 1: Install Sealed Secrets

Sealed Secrets provides a secure and declarative way to manage Kubernetes secrets.

Install the controller in the cluster:

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.26.0/controller.yaml
```

Verify that the controller is running:

```bash
kubectl wait --for=condition=available deployment/sealed-secrets-controller -n kube-system --timeout=600s
```

Install kubeseal on your machine:

```bash
kubeseal --version
```

---

## 🔄 Step 2: Install Argo CD

This repository already includes declarative manifests for Argo CD under [infra/kubernetes/base/argocd](infra/kubernetes/base/argocd).

Install it with:

```bash
kubectl apply -k infra/kubernetes/base/argocd
```

Wait for the Argo CD server to become available:

```bash
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=600s
```

Access the UI locally:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then open:

```text
https://localhost:8080
```

The initial admin credentials are:

- username: `admin`
- password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
```

The value returned is base64-encoded. Decode it with the tools available in your operating system and change the password after the first login.

---

## 🗝️ Step 3: Use Sealed Secrets in this project

This repository includes a secret template and example values under [infra/kubernetes/base/secrets](infra/kubernetes/base/secrets).

The intended workflow is:

1. keep a Secret template in the repository;
2. fill it with local values only when needed;
3. seal it with `kubeseal`;
4. commit the resulting SealedSecret manifest;
5. let Argo CD apply it to the cluster.

Example:

```bash
kubectl create secret generic example-secret --from-literal=foo=bar --dry-run=client -o yaml > secret.yaml
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
```

This keeps secrets encrypted and safe to commit while still being usable by Kubernetes.

---

## 🖥️ How this project is intended to run locally

Once the cluster is up and the base components are installed, Argo CD will be used to apply the remaining infrastructure and application manifests for:

- 🗄️ MinIO
- 🌬️ Airflow
- ⚡ Spark Operator
- 🧱 the lakehouse pipeline components

The local development flow is therefore:

1. start Docker Desktop and enable Kubernetes;
2. ensure `kubectl` points to the local cluster;
3. install Sealed Secrets;
4. install Argo CD;
5. apply the declarative manifests from this repository;
6. let Argo CD keep the environment synchronized.

This approach makes the setup reproducible and easy to replicate on another machine.

---

## 📚 Documentation index

### 🏗️ Infrastructure

- [infra/README.md](infra/README.md) — infrastructure directory overview
- [infra/kubernetes/README.md](infra/kubernetes/README.md) — Kubernetes directory overview
- [infra/kubernetes/base/README.md](infra/kubernetes/base/README.md) — base manifests overview
- [infra/kubernetes/base/argocd/README.md](infra/kubernetes/base/argocd/README.md) — Argo CD installation and access
- [infra/kubernetes/base/sealed-secrets/README.md](infra/kubernetes/base/sealed-secrets/README.md) — Sealed Secrets controller and workflow
- [infra/kubernetes/base/secrets/README.md](infra/kubernetes/base/secrets/README.md) — secret templates and sealing workflow
- [infra/kubernetes/overlays/README.md](infra/kubernetes/overlays/README.md) — environment-specific overlays
- [infra/kubernetes/charts/README.md](infra/kubernetes/charts/README.md) — reusable Helm charts
- [infra/terraform/README.md](infra/terraform/README.md) — Terraform provisioning (future)

### 🧱 Pipelines and source code

- [src/README.md](src/README.md) — source code overview
- [src/pipelines/README.md](src/pipelines/README.md) — pipeline stages overview
- [src/pipelines/bronze/README.md](src/pipelines/bronze/README.md) — 🥉 bronze ingestion layer
- [src/pipelines/silver/README.md](src/pipelines/silver/README.md) — 🥈 silver deduplication and consolidation layer
- [src/pipelines/gold/README.md](src/pipelines/gold/README.md) — 🥇 gold feature and analytics layer
- [src/shared/README.md](src/shared/README.md) — shared utilities and helpers

### ⚙️ Orchestration and operations

- [dags/README.md](dags/README.md) — Airflow DAGs
- [scripts/README.md](scripts/README.md) — operational scripts
- [tests/README.md](tests/README.md) — tests

### 📖 Project context

- [CLAUDE.md](CLAUDE.md) — full project context, architecture decisions and data source inventory
