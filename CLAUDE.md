# CLAUDE.md — Homelab Cluster

## Rules for Claude

- **Never commit or push changes.** Only edit files. The user handles all git commits and pushes.

## Project Overview

This is a **GitOps-managed Kubernetes cluster repository** using **Flux CD v2**. All cluster state is declared in this repository and continuously reconciled by Flux running inside the cluster. The target environment is a single staging cluster.

## Repository Layout

```
homelab-cluster/
├── clusters/staging/          # Flux entry point — root Kustomization
│   ├── flux-system/           # Flux operator manifests and sync config
│   ├── apps.yaml              # Kustomization for apps/staging
│   └── monitoring.yaml        # Kustomization for monitoring stack
├── apps/
│   ├── base/                  # Reusable Kustomize bases (no env-specific values)
│   │   ├── churlesflows/      # n8n + PostgreSQL + Redis base configs
│   │   └── linkding/          # Linkding base config
│   └── staging/               # Staging overlays (cloudflare tunnels, ingress, env secrets)
│       ├── churlesflows/
│       └── linkding/
├── monitoring/
│   ├── controllers/base/      # kube-prometheus-stack HelmRelease base
│   └── configs/staging/       # Grafana secrets (SMTP, TLS)
└── .devcontainer/             # Dev environment (Ubuntu jammy, GitHub CLI, Node, 1Password CLI)
```

## GitOps Workflow

- **Flux** watches the `main` branch of this repo every **1 minute**.
- Changes merged to `main` are automatically reconciled into the cluster.
- `prune: true` is set — removing a resource from Git removes it from the cluster.
- Never apply manifests directly with `kubectl apply` unless debugging; commit to Git instead.

## Secret Management

All secrets use **SOPS with Age encryption** and are committed encrypted to the repository.

- Age public key: `age1c8uz8qzz095uspf99rfpketpcl9fk3zpwtjkts0nx4tyxctjqdyspqzj36`
- SOPS config: `clusters/staging/.sops.yaml`
- Rule: encrypt only `data` and `stringData` fields (regex: `^(data|stringData)$`)
- Flux decrypts automatically at reconcile time using the `sops-age` secret in `flux-system`

**To encrypt a new secret:**
```bash
sops --encrypt --in-place <secret-file>.yaml
```

**Never commit plaintext secrets.** All `*-secret.yaml` files inside `apps/base/` and `monitoring/configs/` must be SOPS-encrypted before pushing.

## Naming Conventions

| Resource | Pattern | Example |
|---|---|---|
| Namespaces | app-purpose | `churlesflows`, `linkding`, `monitoring` |
| Secrets | `{app}-{purpose}-secret` | `postgres-churlesflow-secret` |
| HelmReleases | match chart name | `postgresql`, `redis`, `n8n` |
| Ingress hostnames | `{short}.experientialabs.net` | `grs.experientialabs.net` |

## File Naming Conventions Per Directory

Each application directory uses consistent filenames:
- `namespace.yaml` — Namespace
- `repository.yaml` — HelmRepository source
- `helmrelease.yaml` — HelmRelease with value overrides
- `deployment.yaml` — Direct Deployment (used for apps without Helm charts, e.g. linkding)
- `service.yaml` — ClusterIP Service
- `storage.yaml` — PersistentVolumeClaim
- `ingress.yaml` — Ingress (staging overlay only)
- `cloudflare.yaml` — Cloudflared Deployment + ConfigMap (staging overlay only)
- `secrets/` — SOPS-encrypted Secret files

## Deployed Applications

| App | Namespace | Access | Helm Chart |
|---|---|---|---|
| n8n (workflow automation) | `churlesflows` | `flows.experientialabs.net` | `oci://8gears.container-registry.com/library/n8n` |
| PostgreSQL | `churlesflows` | internal | `oci://registry-1.docker.io/bitnamicharts/postgresql` |
| Redis | `churlesflows` | internal | `oci://registry-1.docker.io/bitnamicharts/redis` |
| Linkding (bookmarks) | `linkding` | `ldng.experientialabs.net` / `ld.experientialabs.net` | none (direct Deployment) |
| kube-prometheus-stack | `monitoring` | `grs.experientialabs.net` | `prometheus-community/kube-prometheus-stack` |

## External Access Architecture

- **Cloudflare Tunnels** — Primary external access, runs `cloudflared` as an in-cluster Deployment
  - `flows.experientialabs.net` → n8n (port 5678)
  - `ldng.experientialabs.net` → linkding (port 9090)
- **Traefik Ingress** — Secondary ingress controller
  - `ld.experientialabs.net` → linkding
  - `grs.experientialabs.net` → Grafana (TLS)

## Monitoring Stack

- **Prometheus + Grafana + Alertmanager** via `kube-prometheus-stack` Helm chart
- Grafana admin user: `canriquez`
- SMTP alerts via **Brevo** (`smtp-relay.brevo.com:587`, from `a59261001@smtp-brevo.com`)
- Redis ServiceMonitor enabled (30s scrape interval, label: `monitoring` namespace)

## Development Environment

The `.devcontainer/` provides a ready-to-use environment with:
- **GitHub CLI** (`gh`)
- **Node.js**
- **1Password CLI** (`op`) — for managing credentials locally
- **Vim** (customized)

## Common Tasks

### Adding a new application
1. Create `apps/base/<app-name>/` with base manifests
2. Create `apps/staging/<app-name>/` overlay (cloudflare, ingress, env-specific patches)
3. Add the staging path to `clusters/staging/apps.yaml` Kustomization
4. Encrypt any secrets with SOPS before committing

### Rotating a secret
1. Decrypt the secret: `sops --decrypt <file>.yaml`
2. Update the value
3. Re-encrypt: `sops --encrypt --in-place <file>.yaml`
4. Commit and push — Flux will reconcile automatically

### Updating a Helm chart version
Edit the `spec.chart.spec.version` field in the relevant `helmrelease.yaml` and commit to `main`.

### Checking Flux reconciliation status
```bash
flux get kustomizations
flux get helmreleases -A
flux logs --follow
```

## Documentation Strategy

All documentation lives under `docs/` and is organized by domain. When creating new docs, follow this structure:

```
docs/
├── monitoring/                  # Grafana, Prometheus, alerting
│   ├── grafana-smtp.md          # SMTP / email notification setup
│   └── dashboards/              # One file per Grafana dashboard
│       └── rayces-prd-overview.md
├── apps/                        # Per-application operational docs
│   └── <app-name>/
├── deployments/                 # Point-in-time deployment reports
│   └── rayces-production-initial-deploy.md
└── infrastructure/              # Cluster-level concerns (ingress, secrets, DNS)
```

### Rules for docs

- **One file per logical topic.** Don't mix unrelated concerns in one file.
- **Dashboards get their own file** under `docs/monitoring/dashboards/<dashboard-uid>.md`.
- **Each doc must include:** overview, architecture or context, configuration details, and a troubleshooting section where relevant.
- **Cross-reference related docs** using relative Markdown links.
- **Grafana dashboards provisioned manually** (not via ConfigMap) must be documented here since they are not version-controlled elsewhere.
- When a new service, integration, or Grafana dashboard/alert set is configured, create or update the corresponding doc in `docs/`.

---

## Key Constraints

- **Single environment:** Only `staging` exists. Do not create `prod/` or `dev/` paths without discussion.
- **Encrypted secrets only:** Never push plaintext secrets. CI will not catch this automatically — it is a manual discipline.
- **Prune is enabled:** Removing a file from the repo removes the resource from the cluster. Be intentional about deletions.
- **Flux manages everything:** Avoid one-off `kubectl apply` — it creates drift and will be overwritten on next reconcile.
