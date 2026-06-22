# Homelab Cluster — GitOps with Flux CD

A **GitOps-managed Kubernetes cluster** where all state is declared in this repository and
continuously reconciled by **[Flux CD v2](https://fluxcd.io/)** running inside the cluster.
Secrets are encrypted at rest with **SOPS + Age**. The target environment is a single
`staging` cluster.

> Push to `main` → Flux reconciles the cluster within ~1 minute. Removing a file removes the
> resource from the cluster (`prune: true`). Avoid one-off `kubectl apply` — it creates drift.

---

## How it works

- **Source:** Flux watches the `main` branch of `ssh://git@github.com/canriquez/homelab-cluster`.
- **Entry point:** the root Flux `Kustomization` (`flux-system`) reconciles `./clusters/staging`,
  which in turn fans out into the app, monitoring, and database stacks.
- **Decryption:** Flux decrypts SOPS-encrypted secrets at reconcile time using the `sops-age`
  secret in the `flux-system` namespace.

```
clusters/staging/
├── flux-system/        # Flux components + GitRepository/Kustomization sync config
├── apps.yaml           # Kustomization → ./apps/staging
├── monitoring.yaml     # Kustomizations → ./monitoring/controllers + ./monitoring/configs
└── database.yaml       # Kustomizations → CloudNativePG operator + dev/staging/prod instances
```

---

## Repository layout

```
homelab-cluster/
├── clusters/staging/              # Flux entry point (root Kustomizations + sync config)
├── apps/
│   ├── base/                      # Reusable Kustomize bases (env-agnostic)
│   │   ├── churlesflows/          # n8n + PostgreSQL + Redis
│   │   └── linkding/              # Linkding (direct Deployment, no Helm)
│   └── staging/                   # Staging overlays (cloudflare tunnels, ingress, secrets)
│       ├── churlesflows/
│       ├── linkding/
│       └── rayces/                # RayCES app (external Helm chart + image automation)
├── database/                      # CloudNativePG (CNPG) PostgreSQL platform
│   ├── controllers/staging/       # CNPG operator HelmRelease + CloudBeaver UI
│   ├── instances/{dev,staging,prod}/   # One Postgres Cluster per environment
│   └── app-secrets/{dev,staging,prod}/ # Connection secrets projected into app namespaces
├── monitoring/
│   ├── controllers/base/          # kube-prometheus-stack HelmRelease base
│   └── configs/staging/           # Grafana admin/SMTP/TLS secrets + ServiceMonitors
├── docs/                          # Operational documentation (see index below)
└── .devcontainer/                 # Dev environment (GitHub CLI, Node, 1Password CLI, Vim)
```

---

## Deployed workloads

| Workload | Namespace | Source | External access |
|---|---|---|---|
| **n8n** (workflow automation) | `churlesflows` | Helm `n8n` (8gears OCI) | `flows.experientialabs.net` (Cloudflare tunnel) |
| **PostgreSQL** (n8n backend) | `churlesflows` | Helm `postgresql` 16.7.x (Bitnami OCI) | internal |
| **Redis** (n8n queue) | `churlesflows` | Helm `redis` 18.1.x (Bitnami OCI) | internal |
| **Linkding** (bookmarks) | `linkding` | direct Deployment | `ldng.` (tunnel) / `ld.` (Traefik) |
| **RayCES** | `rayces-production` | external Git Helm chart + image automation | Cloudflare tunnel |
| **CloudNativePG operator** | `cnpg-system` | Helm `cloudnative-pg` 0.22.x | — |
| **RayCES databases** | `rayces-db-{dev,staging,prod}` | CNPG `Cluster` (PostgreSQL 16.4) | internal |
| **CloudBeaver** (DB UI) | `cnpg-system` | direct Deployment | internal |
| **kube-prometheus-stack** | `monitoring` | Helm `kube-prometheus-stack` 72.6.x | `grs.experientialabs.net` (Traefik, TLS) |

---

## Application stacks

### churlesflows (n8n)
n8n workflow automation backed by Bitnami PostgreSQL and Redis. The base
(`apps/base/churlesflows/`) holds the HelmReleases, repositories, and SOPS-encrypted secrets;
the staging overlay adds the Cloudflare tunnel and ingress. PostgreSQL credentials are supplied
to the chart via `valuesFrom` an encrypted `values.yaml` secret.

### linkding
A self-hosted bookmark manager deployed as a plain `Deployment` (no Helm chart) with a PVC for
persistence. Exposed both through a Cloudflare tunnel (`ldng.`) and a Traefik `Ingress` (`ld.`).

### rayces
An external application: the Helm chart is pulled from a **separate Git repository**
(`github.com/canriquez/rayces-v3`, path `charts/rayces`) and deployed into `rayces-production`.
Flux **image automation** watches the `rayces-backend` / `rayces-frontend` GHCR image
repositories and automatically commits updated image tags back to this repo (the recurring
`chore: update rayces images` commits). RayCES connects to its dedicated CNPG database via the
connection secrets in `database/app-secrets/`.

---

## Database platform (CloudNativePG)

The CNPG operator manages a dedicated PostgreSQL `Cluster` per environment for RayCES:

| Environment | Cluster / namespace | Database | Backups |
|---|---|---|---|
| dev | `rayces-db-dev` | `raycesv3_development` | — |
| staging | `rayces-db-staging` | `raycesv3_staging` | WAL + base → S3-compatible store |
| prod | `rayces-db-prod` | `raycesv3_production` | continuous WAL + hourly base → Cloudflare R2 (PITR, 30d) |

- All clusters run `ghcr.io/cloudnative-pg/postgresql:16.4`, owner role `raycesv3`.
- The operator (`database-controllers`) is reconciled first; each instance `dependsOn` it.
- Connection details for consuming apps live in `database/app-secrets/<env>/connection-secret.yaml`
  (encrypted) and are projected into the application namespaces.
- **CloudBeaver** provides a web DB client inside the cluster for inspecting these databases.

See [docs/deployments/rayces-db-prod-cnpg.md](docs/deployments/rayces-db-prod-cnpg.md) and the
[CNPG migration case study](docs/cnpg-database-migration-case-study.md) for details.

---

## Monitoring

`kube-prometheus-stack` provides Prometheus + Grafana + Alertmanager in the `monitoring`
namespace. Grafana is exposed at `grs.experientialabs.net` (Traefik, TLS) and sends email
alerts through Brevo SMTP. Grafana admin credentials, SMTP credentials, and TLS are stored as
separate SOPS-encrypted secrets and wired into the chart:

- **Admin credentials** → `grafana-admin-credentials` via the chart's `admin.existingSecret`.
- **SMTP credentials** → `grafana-smtp-credentials` via `envFromSecret` + `grafana.ini` `$__env{...}`.

See [docs/monitoring/grafana-smtp.md](docs/monitoring/grafana-smtp.md).

---

## External access

- **Cloudflare Tunnels** (primary) — `cloudflared` runs as in-cluster Deployments:
  - `flows.experientialabs.net` → n8n
  - `ldng.experientialabs.net` → linkding
  - RayCES via its own tunnel
- **Traefik Ingress** (secondary):
  - `ld.experientialabs.net` → linkding
  - `grs.experientialabs.net` → Grafana (TLS)

---

## Secret management (SOPS + Age)

All secrets are committed **encrypted**. Only `data` / `stringData` fields are encrypted
(regex `^(data|stringData)$`); key names stay readable.

- Age recipient: `age1c8uz8qzz095uspf99rfpketpcl9fk3zpwtjkts0nx4tyxctjqdyspqzj36`
- SOPS config: [.sops.yaml](.sops.yaml) (and per-tree overrides such as `clusters/staging/.sops.yaml`)

**Never commit plaintext secrets** — CI does not enforce this; it is a manual discipline.

Encrypt a new secret:
```bash
sops --age=$AGE_PUBLIC --encrypt --encrypted-regex '^(data|stringData)$' --in-place <secret-file>.yaml
```

Rotate an existing secret:
```bash
sops --decrypt --in-place <file>.yaml     # edit the value
sops --encrypt --in-place <file>.yaml      # re-encrypt; commit — Flux reconciles
```

---

## Common operations

```bash
# Reconciliation status
flux get kustomizations
flux get helmreleases -A
flux logs --follow

# Force a reconcile after pushing
flux reconcile kustomization flux-system --with-source

# Inspect a CNPG cluster
kubectl get cluster -n rayces-db-prod
kubectl get scheduledbackup,backup -n rayces-db-prod
```

Updating a Helm chart version: edit `spec.chart.spec.version` in the relevant `helmrelease.yaml`
(or `release.yaml`) and commit to `main`.

---

## Documentation

| Doc | Topic |
|---|---|
| [docs/deployments/rayces-db-prod-cnpg.md](docs/deployments/rayces-db-prod-cnpg.md) | RayCES production CNPG database deployment report |
| [docs/cnpg-database-migration-case-study.md](docs/cnpg-database-migration-case-study.md) | Migration to CloudNativePG — case study |
| [docs/guides/postgres-backup-gcs.md](docs/guides/postgres-backup-gcs.md) | PostgreSQL backups to object storage |
| [docs/monitoring/grafana-smtp.md](docs/monitoring/grafana-smtp.md) | Grafana SMTP / email alerting via Brevo |
| [docs/monitoring/dashboards/rayces-prd-overview.md](docs/monitoring/dashboards/rayces-prd-overview.md) | RayCES production Grafana dashboard |

Repository conventions and detailed task playbooks live in [CLAUDE.md](CLAUDE.md).

---

## Key constraints

- **Single environment** for cluster bootstrap: only `clusters/staging` exists. (The CNPG
  database platform additionally defines `dev`/`staging`/`prod` instances.)
- **Encrypted secrets only** — never push plaintext.
- **Prune is enabled** — deleting a file deletes the cluster resource. Be intentional.
- **Flux manages everything** — commit to Git instead of applying manifests directly.
