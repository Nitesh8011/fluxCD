# fluxCD

Learning FluxCD from scratch — GitOps continuous delivery for Kubernetes using this repo as the source of truth.

## Status

- ✅ Kustomization chapter — done
- ✅ Helm chapter — nginx-app chart wired up for staging + production
- ✅ Monitoring chart — single-pod Prometheus/Loki/Grafana chart wired to staging only, for testing
- ⏭️ Jenkins CI — Jenkinsfile added, needs a running Jenkins to wire up
- ⏭️ Flux controller monitoring (dashboards/alerts on Flux itself) — not started, see Roadmap

## Repo layout

```text
clusters/
  staging/
    flux-system/         # Flux-managed bootstrap manifests (gotk-components, gotk-sync)
    staging-kustomize.yaml   # points at flux/kustomize/staging
    staging-helm.yaml        # points at flux/helm/releases/staging
    staging-monitoring.yaml  # points at flux/helm/releases/monitoring (staging only)
  production/
    flux-system/
    production-kustomize.yaml  # points at flux/kustomize/production
    production-helm.yaml       # points at flux/helm/releases/production

flux/kustomize/
  base/                  # shared deployment + service
  staging/                # staging overlay (namespace, configmap)
  production/             # production overlay (namespace, configmap)

flux/helm/
  nginx-app/              # Helm chart (Chart.yaml, templates/, values.yaml + per-env values files)
  releases/
    staging/               # HelmRelease pointing at nginx-app + values-staging.yaml
    production/             # HelmRelease pointing at nginx-app + values-production.yaml
    monitoring/              # HelmRelease pointing at flux/monitoring, targetNamespace flux-monitoring-ns

flux/monitoring/          # Umbrella Helm chart: single-pod Prometheus + Loki + Grafana, for testing only

Jenkinsfile               # CI: helm lint, helm template, kustomize build (see below)
```

Each cluster's Kustomization under `clusters/<env>` references the matching overlay in `flux/kustomize/<env>` (built on `flux/kustomize/base`) or the matching HelmRelease in `flux/helm/releases/<env>` (built on the shared `flux/helm/nginx-app` chart). The monitoring stack is currently wired to staging only, via `clusters/staging/staging-monitoring.yaml`.

## Helm chart gotchas (learned the hard way)

- `flux/helm/nginx-app/values.yaml` is the chart's **default** values, always loaded first by Helm even though no `HelmRelease` references it directly. The per-env `values-staging.yaml` / `values-production.yaml` only override what they set — anything else still comes from `values.yaml`.
- The `HelmChart` Flux generates for a git-sourced chart defaults to `reconcileStrategy: ChartVersion`. That means editing a values file (or any file under the chart path) does **not** trigger a repackage/upgrade on its own — only bumping `Chart.yaml`'s `version:` does. Forgot this once and staging kept deploying stale `replicaCount` values even after the git commit was correct.
- Both `staging` and `production` HelmReleases track the same `master` branch and the same chart path, so there's only ever one `Chart.yaml` version live for both environments at a time — you can't pin prod to an older chart version without pointing its `GitRepository` at a different ref/tag.

## CI (Jenkins)

A `Jenkinsfile` at the repo root runs on every change:

- `helm lint` on `flux/helm/nginx-app`
- `helm template` against both `values-staging.yaml` and `values-production.yaml`
- `kubectl kustomize` build for every kustomize overlay and cluster path

It only validates that charts/overlays render cleanly — no cluster access, no deploys. Point a Jenkins multibranch pipeline at this repo to pick it up.

## Installing the monitoring stack (staging)

The `flux/monitoring` chart (Prometheus + Loki + Grafana, single pod each, no persistence) is wired
into staging via `clusters/staging/staging-monitoring.yaml` → `flux/helm/releases/monitoring/helmrelease.yaml`.
It's not on production.

1. Commit and push the changes:

   ```bash
   git add flux/monitoring flux/helm/releases/monitoring clusters/staging/staging-monitoring.yaml clusters/staging/kustomization.yaml
   git commit -m "Wire up monitoring stack on staging"
   git push
   ```

2. Force Flux to pick it up immediately instead of waiting on the sync interval:

   ```bash
   flux reconcile source git flux-system --context=staging
   flux reconcile kustomization flux-system --context=staging --with-source
   flux reconcile kustomization apps-monitoring --context=staging
   ```

3. Watch it come up:

   ```bash
   flux get helmrelease monitoring -n flux-system --context=staging
   kubectl get pods -n flux-monitoring-ns --context=staging
   ```

`spec.install.createNamespace: true` on the `HelmRelease` handles creating `flux-monitoring-ns` — the
chart itself has no `namespace.yaml` template (unlike `nginx-app`).

## Roadmap

- **Flux monitoring**: expose Flux controller health/drift to something observable — options considered: Prometheus `ServiceMonitor`s for the `flux-system` controllers + a Grafana dashboard, or a simpler scheduled job running `flux check` / `flux get all -A` and alerting on failure. Not implemented yet, revisit once a monitoring stack exists in-cluster.
- **CI schema validation**: extend the Jenkinsfile with `kubeconform`/`kubeval` to schema-check rendered manifests, not just confirm they render.

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
