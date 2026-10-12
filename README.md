# Azure DevOps Pipeline Templates — ACR → Helm → AKS

Reusable Azure DevOps YAML pipelines: build a Docker image, gate it on a Trivy
vulnerability scan, push it to Azure Container Registry, then deploy it with Helm
to AKS across dev and prod — with a manual approval gate on prod and a Dynatrace
deployment event posted after every release.

## Pipeline flow

```
main push ─┐
           ├─► [Build]        docker build (BuildKit) → Trivy scan (fail on HIGH/CRITICAL)
PR ────────┘         │
                     ▼  (main branch only)
              [Push]          push image → ACR
                     │
                     ▼
              [Deploy-Dev]    Helm upgrade → dev namespace → Dynatrace deployment event
                     │
                     ▼  (manual approval on the prod environment)
              [Deploy-Prod]   Helm upgrade → prod namespace → Dynatrace deployment event
```

- **PRs** run Build only (build + scan, no push) — fast feedback, nothing published.
- **Prod deploys are approval-gated** via an Azure DevOps environment check, not pipeline YAML.

## Repository layout

```
azure-pipelines.yml            # main multi-stage pipeline
templates/
  docker-build-push.yml        # reusable: build → Trivy scan → push to ACR
  helm-deploy.yml              # reusable: Helm upgrade --install --atomic onto AKS
  dynatrace-event.yml          # reusable: post CUSTOM_DEPLOYMENT event to Dynatrace
docker/
  Dockerfile                   # sample app (unprivileged nginx)
  index.html
helm/                          # sample Helm chart (deployable as-is)
  values.yaml / values-dev.yaml / values-prod.yaml
  templates/                   # deployment (probes), service, ingress (TLS), HPA, PDB
docs/
  variables.md                 # variable groups, secrets, service connections
```

## Prerequisites

- Azure DevOps project with Pipelines enabled
- Azure Container Registry (ACR)
- AKS cluster
- Service connections (see `docs/variables.md`):
  - **Docker Registry** connection backed by ACR (build/push)
  - **Azure Resource Manager** connection for the subscription hosting AKS (deploy)
- Variable groups `sample-webapp-common` and `sample-webapp-secrets` (see `docs/variables.md`)

## Wire it to a new application (3 steps)

1. **Copy the pipeline files** into your app repo: `azure-pipelines.yml`, `templates/`, and either the
   sample `helm/` chart or your own (point `chartPath` at it in the deploy stages).
2. **Create the variable groups and service connections** per `docs/variables.md`, then update the
   `variables:` block and the `acrServiceConnection` value in `azure-pipelines.yml`.
3. **Create the environments** `sample-webapp-dev` and `sample-webapp-prod`
   (Pipelines > Environments), then add a manual **Approvals** check on the prod environment —
   that is what gates `Deploy_Prod`. Rename the environments per app and update the pipeline.

## Dynatrace setup

1. **OneAgent (deep monitoring, optional):** install the Dynatrace Operator in the AKS cluster
   (see Dynatrace Hub: "Dynatrace Operator"). Then set `dynatrace.enabled: true` in your values
   file — pods get the `dynatrace.com/inject` annotation for automatic OneAgent injection.
2. **Deployment events (included):** create an API token at
   Settings > Integration > Dynatrace API with the **`events.ingest`** scope, store it as the
   `DynatraceApiToken` secret variable, and set `dynatraceUrl` in the common variable group.
   Every deploy stage posts a `CUSTOM_DEPLOYMENT` event with version + environment, so releases
   appear on the Dynatrace timeline next to your metrics.

## Production hardening included

- Liveness + readiness probes on every pod
- PodDisruptionBudget (`minAvailable: 1`) so voluntary disruptions never take the app fully down
- HPA (2–6 replicas, 70% CPU target); static `replicas` omitted when autoscaling is on
- Ingress with `ingressClassName` parameter and a TLS block
- Resource requests/limits on all containers
- Trivy image scan gates the push — HIGH/CRITICAL vulnerabilities fail the build
- `--atomic` Helm upgrades (automatic rollback on failure) with `--create-namespace`
- Non-root sample container (`nginx-unprivileged`)
- Secrets stay in Azure DevOps variable groups / service connections — nothing in the repo
