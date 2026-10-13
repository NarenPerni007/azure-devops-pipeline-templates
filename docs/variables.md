# Variable groups & service connections

Create these once per application (Pipelines > Library > Variable groups, and
Project Settings > Service connections), then point the pipeline at them.

## Variable group: `sample-webapp-common` (plain variables)

| Variable | Example | Purpose |
|---|---|---|
| `acrName` | `myacr` | ACR name **without** `.azurecr.io`; used to tag/push images |
| `azureSubscription` | `My Azure subscription` | ARM service connection name for the subscription hosting AKS |
| `aksResourceGroup` | `rg-aks-prod` | Resource group of the AKS cluster |
| `aksClusterName` | `aks-prod` | AKS cluster name |
| `dynatraceUrl` | `https://abc123.live.dynatrace.com` | Dynatrace environment URL (no trailing slash) |
| `devNamespace` | `sample-webapp-dev` | Kubernetes namespace for dev |
| `prodNamespace` | `sample-webapp-prod` | Kubernetes namespace for prod |

## Variable group: `sample-webapp-secrets` (secret variables)

| Variable | Purpose |
|---|---|
| `DynatraceApiToken` | Dynatrace API token with the **`events.ingest`** scope. Mark as secret. Passed to `templates/dynatrace-event.yml` as `'$(DynatraceApiToken)'`. |

> Rename the groups per app (e.g. `orders-api-common`) and update the `variables:` block
> in `azure-pipelines.yml` to match.

## Service connections

| Name (example) | Type | Used by |
|---|---|---|
| `acr-docker-sc` | **Docker Registry** (backed by ACR) | `templates/docker-build-push.yml` - login + push |
| `My Azure subscription` | **Azure Resource Manager** (service principal) | `templates/helm-deploy.yml` - AKS cluster access for `HelmDeploy@1` |

Both are referenced by **name** in the pipeline - no credentials live in the repo.
