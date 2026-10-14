# PillGuide GitOps

Argo CD tracks **main** in this repository. `argocd/root.yaml` is the bootstrap Application; it manages `argocd/apps/`.

| Backend branch | Manifest on main | Argo CD Application / namespace | API |
|---|---|---|---|
| develop | dev/deployment.yaml | dev | https://dev-api.pillguide.site |
| main | production/deployment.yaml | production | https://api.pillguide.site |

Backend CI publishes a Linux ARM64 image to `wien0128/pillguide-be` and updates only the matching Deployment image by immutable digest. Both environments share this repository's main branch, not separate Git branches.

## Spring deployments

Both dev and production currently declare `replicas: 1` and use immutable ARM64 image digests. CI updates image digests, not replica counts.

Change desired replica counts in this repository. Do not scale only through kubectl: Argo CD self-healing restores the Git value.

The older `pillguide/` MySQL and Deployment manifests are retired. The old `codex/pillguide-gitops` branch is retained for history; no Git Application should track it. The external-secrets Helm Application remains pinned to its chart version.

## Shared inference DNS

`argocd/apps/platform-dns.yaml` manages `platform/coredns/config-map.yaml` in `kube-system`. The root Application discovers this child through its existing recursive `argocd/apps/` source. Keep this shared ConfigMap out of the dev and production directories to avoid duplicate ownership.

The ConfigMap uses K3s's existing `/etc/coredns/custom/*.server` import. It maps `desktop-77j9ol4.taildd6361.ts.net` to `100.121.91.96`, with a record TTL of 60 seconds and cache of 30 seconds. It preserves the validated DNS data without replacing K3s's managed Corefile or changing other DNS zones. The existing same-name ConfigMap can be adopted without deleting it first.

Automatic sync and self-healing are enabled; automatic pruning is disabled. The child has no cascade finalizer, matching the other Applications. Removing the child Application manifest from Git lets the root prune the Application while leaving the DNS ConfigMap in place. Removing only the ConfigMap manifest also leaves the live ConfigMap because this child's automatic pruning is disabled. Intentional resource removal requires a separate reviewed operation. Roll back DNS data through Git while the Application is managed, since manual changes would be self-healed.

This is a static mapping. Update it if the desktop's Tailscale address changes. It does not configure the EC2 Tailscale connection, access policy, desktop Serve process, inference-server startup, or API credentials. Do not put API keys, private keys, or other credentials in these manifests or verification output.

## Cluster rebuild and verification

1. Restore K3s, Argo CD, repository access, and existing runtime prerequisites through the approved bootstrap procedure. Confirm K3s still mounts `coredns-custom` and imports `*.server` files. Restore EC2 Tailscale connectivity and the approved desktop inference/Serve processes separately.
2. Bootstrap the root with `kubectl apply -f argocd/root.yaml`. Its existing auto-sync creates the child Applications from main, including `platform-dns`.
3. Confirm all six Applications are Synced/Healthy: `pillguide-root`, `argocd-config`, `external-secrets`, `platform-dns`, `dev`, and `production`. Check that CoreDNS has loaded the custom mapping. Allow time for the ConfigMap volume update and CoreDNS reload; do not assume a Pod restart is required.
4. From each actual dev and production backend container, verify the desktop FQDN resolves to `100.121.91.96`, then call authenticated `/health` and a sample `/predict` using the normal HTTPS hostname and certificate validation. Supply credentials through a secure input mechanism, not command arguments or Git. Compare the sample with the saved model H output, using the established absolute tolerance of 0.001.
5. From both containers, also verify Kubernetes service, AWS, and RDS name resolution and confirm backend/CoreDNS readiness and restart counts are unchanged by DNS adoption.

The child Application uses sync wave 0, but this alone does not guarantee its DNS configuration is ready before other child workloads. Synced/Healthy is not an end-to-end inference check; the Pod-level checks above are the completion criteria.

Merging changes into main can trigger live reconciliation. Local edits or a review branch do not deploy this configuration. Backend code and automatic-start settings are outside this change.

References: [K3s custom CoreDNS imports](https://docs.k3s.io/advanced#coredns-custom-configuration-imports), [Argo CD automatic sync](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/), [Application health and sync waves](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/).
