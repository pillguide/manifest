# PillGuide GitOps

Argo CD tracks **main** in this repository. `argocd/root.yaml` is the bootstrap Application; it manages `argocd/apps/`.

| Backend branch | Manifest on main | Argo CD Application / namespace | API |
|---|---|---|---|
| develop | dev/deployment.yaml | dev | https://dev-api.pillguide.site |
| main | production/deployment.yaml | production | https://api.pillguide.site |

Backend CI publishes a Linux ARM64 image to `wien0128/pillguide-be` and updates only the matching Deployment image by immutable digest. Both environments share this repository's main branch, not separate Git branches.

## First Spring rollout is pending

Both Deployments deliberately have `replicas: 0`. The initial `awaiting-arm64-build` tag is a parking placeholder, not a runnable release. CI must replace it with a successfully built image digest before activation. CI never changes replicas.

Before the separate first rollout: configure Docker Hub credentials in Backend Actions; verify PostgreSQL driver and application configuration, application probes, and OAuth callback URLs. Then change the desired replica count in this repository. Do not scale only through kubectl: Argo CD self-healing restores the Git value.

The older `pillguide/` MySQL and Deployment manifests are retired. The old `codex/pillguide-gitops` branch is retained for history; no Git Application should track it. The external-secrets Helm Application remains pinned to its chart version.

To bootstrap: `kubectl apply -f argocd/root.yaml`. For normal changes, commit to main and let Argo CD reconcile. Confirm all five Applications are Synced/Healthy and that dev/production have no running Spring Pods until activation.
