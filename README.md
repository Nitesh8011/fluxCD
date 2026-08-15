# fluxCD

Learning FluxCD from scratch — GitOps continuous delivery for Kubernetes using this repo as the source of truth.

## Status

- ✅ Kustomization chapter — done
- ⏭️ Helm chapter — next

## Repo layout

```text
clusters/
  staging/
    flux-system/        # Flux-managed bootstrap manifests (gotk-components, gotk-sync)
    kustomization.yaml   # points at flux/kustomize/staging
    staging.yaml
  production/
    flux-system/
    kustomization.yaml   # points at flux/kustomize/production
    production.yaml

flux/kustomize/
  base/                  # shared deployment + service
  staging/                # staging overlay (namespace, configmap)
  production/             # production overlay (namespace, configmap)
```

Each cluster's `kustomization.yaml` under `clusters/<env>` references the matching overlay in `flux/kustomize/<env>`, which itself builds on `flux/kustomize/base`.

## Flux CLI cheat sheet

```bash
flux --help
```

Main commands:

- `bootstrap` — installs Flux on the cluster, upgrades it to a newer version, and reinstalls extra components if needed.
- `check` — verifies the cluster meets Flux's requirements.
- `create` — creates Flux custom resources locally (typically with `--export`, then commit the output to git).
- `reconcile` — syncs the Flux controllers and updates cluster state on demand.

Full bootstrap reference: <https://fluxcd.io/flux/cmd/flux_bootstrap_github/>

## Local clusters (minikube)

```bash
minikube start -p staging \
  --cpus=2 \
  --memory=4096 \
  --driver=docker

minikube start -p production \
  --cpus=2 \
  --memory=4096 \
  --driver=docker
```

## Bootstrapping Flux

Export a GitHub personal access token first:

```bash
export GITHUB_TOKEN=github_pat_****
```

Staging:

```bash
flux bootstrap github \
  --owner=Nitesh8011 \
  --repository=fluxCD \
  --branch=master \
  --path=./clusters/staging \
  --context=staging \
  --personal
```

Production:

```bash
flux bootstrap github \
  --owner=Nitesh8011 \
  --repository=fluxCD \
  --branch=master \
  --path=./clusters/production \
  --context=production \
  --personal
```
