# Sealed Secrets

This directory holds the Sealed Secrets controller installation and usage notes for the project.

## Installation

The controller is installed directly in the cluster using the official manifest:

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.26.0/controller.yaml
```

## Client installation

Download the kubeseal binary from the official releases and add it to your PATH.

## Recommended workflow

1. Create a local Kubernetes Secret.
2. Seal it with `kubeseal` using the cluster's public key.
3. Commit the resulting `SealedSecret` manifest to the repository.
4. Argo CD applies it automatically.

## Example

```bash
kubectl create secret generic my-secret --from-literal=foo=bar --dry-run=client -o yaml > secret.yaml
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
```
