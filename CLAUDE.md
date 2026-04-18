# infra

GitOps repository for production server infrastructure.

## Background

Migrated from: nginx (manual config) + certbot + individually-managed docker-compose projects.

## Stack

| Component | Tool |
|---|---|
| Orchestration | k3s (single-node Kubernetes) |
| Reverse proxy + SSL | Traefik (k3s built-in) + cert-manager |
| GitOps | ArgoCD |
| Metrics + dashboards | kube-prometheus-stack (Prometheus + Grafana + Alertmanager) |
| Status page | Gatus |
| Secrets | Sealed Secrets |
| Dependency updates | Renovate |

## Domains + URLs

Domain: `flipez.de`, infrastructure uses `*.cloud.flipez.de`

| Service | URL |
|---|---|
| ArgoCD | `https://argocd.cloud.flipez.de` |
| Grafana | `https://grafana.cloud.flipez.de` |
| Status | `https://status.flipez.de` / `https://status.auch.cool` |

Let's Encrypt email: `letsencrypt@flipez.net`

## Repo layout

```
infra/
│
│  ── "pointers" layer (ArgoCD Application CRDs — no real config here) ──
├── clusters/
│   └── production/
│       ├── argocd/                 ← self-managing ArgoCD Application (applied once manually)
│       ├── apps/                   ← one ArgoCD Application per service (App of Apps)
│       │   └── <name>.yaml
│       ├── infrastructure-app.yaml ← ArgoCD Application: points at infrastructure/
│       └── apps-app.yaml           ← ArgoCD Application: points at clusters/production/apps/
│
│  ── "content" layer (actual manifests applied to the cluster) ──
├── infrastructure/
│   ├── argocd/                     ← ArgoCD ingress + insecure mode config
│   ├── cert-manager/               ← ClusterIssuer (Let's Encrypt)
│   └── monitoring/                 ← kube-prometheus-stack ArgoCD Application
├── apps/
│   └── <name>/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── ingress.yaml
│       ├── sealed-secret.yaml      ← only if the service needs secrets
│       └── pvc.yaml                ← only if the service needs persistent storage
├── temp/                           ← gitignored — plaintext secrets before sealing
│   └── <name>/
│       └── secret.yaml
└── renovate.json                   ← auto PRs for Helm chart + image updates
```

`clusters/` contains only ArgoCD Application objects (pointers). All real config lives in `infrastructure/` and `apps/`.

## Adding a new service

1. Create `apps/<name>/` with `deployment.yaml`, `service.yaml`, `ingress.yaml`
2. If persistent storage is needed, add `pvc.yaml` (see PVC rules below)
3. If secrets are needed, see Secrets section below
4. Add `clusters/production/apps/<name>.yaml` — ArgoCD Application pointing at `apps/<name>/`
5. Add the service to `apps/gatus/configmap.yaml`
6. Push to `main` — ArgoCD auto-syncs

## Ingress + SSL

Every Ingress needs these annotations:

```yaml
annotations:
  cert-manager.io/cluster-issuer: "letsencrypt-prod"
  traefik.ingress.kubernetes.io/router.entrypoints: websecure
```

## Deploying

**Automatic**: push to `main`, ArgoCD syncs within ~3 minutes. Force immediate sync via ArgoCD UI → Refresh.

**Manual**:
```bash
argocd app sync <name>             # sync one service
kubectl apply -f apps/<name>/      # emergency direct apply, bypasses ArgoCD
```

## Persistent volumes

PVCs backed by `local-path` (k3s built-in provisioner).
Data on host: `/var/lib/rancher/k3s/storage/`

**Every PVC must have this annotation** to prevent ArgoCD from ever deleting it:
```yaml
annotations:
  argocd.argoproj.io/sync-options: Prune=false
```

**Every Deployment with a PVC must set `fsGroup`** so Kubernetes fixes ownership automatically on mount:
```yaml
spec:
  template:
    spec:
      securityContext:
        fsGroup: <uid>   # match the container's user — check with: kubectl exec deployment/<name> -- id
```

**Data migration** from old docker-compose path:
```bash
rsync -a /srv/<name>/data/. /var/lib/rancher/k3s/storage/$(ls /var/lib/rancher/k3s/storage/ | grep <pvc-name>)/
```

**PostgreSQL migration**: never rsync a running PostgreSQL — use `pg_dump` instead. If the app already ran migrations on startup, drop and recreate the DB before restoring:
```bash
docker exec <old-container> pg_dump -U postgres <db> > /tmp/dump.sql
kubectl exec deployment/<name>-db -- psql -U postgres -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = '<db>';"
kubectl exec deployment/<name>-db -- psql -U postgres -c "DROP DATABASE <db>;"
kubectl exec deployment/<name>-db -- psql -U postgres -c "CREATE DATABASE <db>;"
kubectl exec -i deployment/<name>-db -- psql -U postgres <db> < /tmp/dump.sql
```

## Secrets

Sealed Secrets is installed. Workflow:

1. Create `temp/<name>/secret.yaml` (gitignored) with plaintext values
2. Seal it: `kubeseal --format yaml < temp/<name>/secret.yaml > apps/<name>/sealed-secret.yaml`
3. Commit `apps/<name>/sealed-secret.yaml` — safe to store in a public repo

Never commit plaintext Kubernetes Secret objects.

## Updating containers

**Pinned tags** (e.g. Synapse): Renovate opens a PR when a new version is available — merge to deploy.

**`latest` tag** (e.g. SpeziDB): `kubectl rollout restart deployment <name>`

## Migrated services

| Service | URL | Notes |
|---|---|---|
| SpeziDB | `spezidb.de` | Rails app, OAuth via Sealed Secrets |
| Matrix | `matrix.auch.cool` | Synapse, pinned to v1.151.0, all config in PVC |
| TheLounge | `irc.auch.cool` | Pinned to 4.4.3 |
| Gatus | `status.flipez.de`, `status.auch.cool` | Status page, replaced Uptime Kuma, SQLite persistence, Telegram alerts on all endpoints |
| Rallly | `rallly.auch.cool` | Scheduling app, PostgreSQL backend, pinned to 4.8.1 |
| Trek | `trek.auch.cool` | Travel tracker, two PVCs (data + uploads), read-only filesystem, pinned to latest |

## Future: NAS cluster

Plan to add a second k3s cluster on the NAS later. Will use:
- Same ArgoCD instance (registered as second cluster via `argocd cluster add`)
- Access via Tailscale (already in use)
- NAS services at `<name>.nas.flipez.de` (private, Tailscale only)
- Existing NAS Prometheus added as a second datasource in cloud Grafana (no separate Grafana on NAS)

---

## Key decisions

- **ArgoCD over Flux**: better web UI for 16+ services; same GitOps guarantees
- **App of Apps pattern**: one ArgoCD Application per service in `clusters/production/apps/` — clean dashboard, separate sync status per service
- **k3s over full k8s**: single binary, SQLite-backed, purpose-built for single-node
- **Traefik (k3s built-in)**: no separate HelmRelease needed; auto-SSL via cert-manager annotations
- **local-path-provisioner**: already ships with k3s, no extra storage setup needed
- **kube-prometheus-stack**: installs Prometheus + Grafana + Alertmanager in one Helm release
- **Gatus over Uptime Kuma**: config-as-code (endpoints in ConfigMap), no UI-only state
- **Sealed Secrets over SOPS**: simpler workflow, well-integrated with ArgoCD
- **Renovate over Dependabot**: Dependabot doesn't understand Helm charts or k8s manifests
- **Public repo**: safe as long as no plaintext secrets are committed
- **`Prune=false` on PVCs**: prevents accidental data loss during ArgoCD restructuring
