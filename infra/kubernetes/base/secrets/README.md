# Secret management

This folder documents the project's secret workflow with Sealed Secrets.

## Purpose

- keep GitOps manifests versioned without storing sensitive values in plain text;
- allow each environment to generate its own SealedSecret locally;
- avoid accidental commits of real secrets.

## Files in this folder

- values.example.env: example file with placeholder values for the required keys;
- sealed-secrets-template.yaml: Secret template used as input for kubeseal;
- .gitignore: ignores local files with real values and generated manifests.

## Recommended workflow

1. Copy values.example.env to a local file such as values.local.env and replace the placeholders with real values.
2. Fill in sealed-secrets-template.yaml with the desired values or use it as a starting point for a temporary manifest.
3. Generate the SealedSecret in the cluster with kubeseal:

```bash
kubeseal --controller-name=sealed-secrets-controller --controller-namespace=kube-system --format yaml \
  -f sealed-secrets-template.yaml > sealed-secret.generated.yaml
```

4. Review the generated manifest and apply it with kubectl or Argo CD.

## Rules

- Never commit values.local.env or generated manifests with real data.
- Keep values.example.env as the public reference file.
- If a new secret is needed, add a new key to the template and the example file.

## Note

The Sealed Secrets controller is running in the kube-system namespace in this local installation.
