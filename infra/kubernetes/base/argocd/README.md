# Argo CD

This directory holds the declarative Argo CD installation for the local Kubernetes cluster.

## Contents

- `namespace.yaml`: creates the `argocd` namespace.
- `install.yaml`: official Argo CD manifest, pinned for reproducibility.
- `kustomization.yaml`: composes the installation declaratively.

## Installation

```bash
kubectl apply -k infra/kubernetes/base/argocd
```

## Local access

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open: https://localhost:8080

Initial credentials:
- username: `admin`
- password (PowerShell):

```powershell
$encoded = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encoded))
```
