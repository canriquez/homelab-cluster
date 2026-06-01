# Case Study: Migrating Rayces to CloudNativePG across dev, staging & prod

> A multi-environment, near-zero-data-loss PostgreSQL migration on a GitOps homelab —
> from a single fragile pod with no backups, to operator-managed clusters with
> point-in-time recovery, executed live with a ~2-minute production window.
>
> **Dates:** 2026-05-30 → 2026-06-01 · **Author:** Carlos Anriquez
> **Repos:** `rayces-v3` (app + Helm chart) and `homelab-cluster` (this repo — Flux GitOps infra).
> This article is mirrored in both. The operational runbook lives in the **`rayces-v3`** repo at
> `docs/runbooks/rayces-db-cnpg.md`.

---

## 1. Executive summary

Rayces is a multi-tenant SaaS for educational/therapeutic services, running on a
self-hosted k3s cluster (`remote-k3s-ubuntu`). Every environment's PostgreSQL was a
**single `postgres:13` pod** on a 500 Mi `local-path` PVC with **`Delete` reclaim policy and
zero backups** — several single actions (a Helm uninstall, a PVC rename, a namespace delete,
or just the node's disk dying) would have irreversibly destroyed customer data, and the app
would have silently re-initialised an empty database on next boot.

We replaced this, across **dev → staging → prod**, with **CloudNativePG (CNPG)**: one
operator-managed Postgres 16 cluster per environment, Flux-managed via GitOps (this repo),
secrets SOPS-encrypted (this repo is public), with **continuous WAL archiving + hourly base
backups to Cloudflare R2** for point-in-time recovery on staging and prod. Each migration was
**blue/green and reversible**; production cut over in a scheduled ~2-minute maintenance window
with zero lost writes.

---

## 2. The starting point

- **Fragility:** single pod, node-local `local-path` PVC, `Delete` reclaim, Helm-owned,
  no `keep` annotation, under a Flux Kustomization with `prune: true`.
- **No backups at all** — not even a nightly dump.
- **Silent failure mode:** the Rails boot command runs `db:prepare`; on an empty volume it
  creates a fresh, empty schema and the app comes up *healthy and blank*.
- **Live stakes:** prod already had real customers configuring booking slots (~14 MB DB).

## 3. Goals & constraints

1. Robust, backed-up Postgres for **all three** environments.
2. **Point-in-time recovery** for staging + prod.
3. **Secrets fully SOPS-encrypted** — this repo is public.
4. **Minimise blast radius:** one environment's mistakes must not touch another's data or
   backups.
5. **Phased & reversible:** dev → staging → prod, old DB kept for rollback, ~2-min window for
   prod only.
6. **Portable:** must lift cleanly into a VPC later.

## 4. Architecture & key decisions

| Decision | Choice & rationale |
|---|---|
| **Engine** | **CloudNativePG** — Apache-2.0, CNCF, the highest-momentum Postgres operator; backup/PITR/HA built in; cloud-agnostic (portability goal). |
| **Topology** | One CNPG `Cluster` per env, each in its own namespace (`rayces-db-{dev,staging,prod}`), each managed by its **own Flux Kustomization** with an independent `prune` scope → cross-env prune is impossible. |
| **Secrets** | Flux/SOPS owns **all** DB credentials, in both the DB and app namespaces. A root `.sops.yaml` (age recipient) encrypts only `data`/`stringData`, so structure stays reviewable. Skaffold no longer mints DB secrets. |
| **App DB role** | A **dedicated non-superuser role `raycesv3`** owns each app database — moving off the `postgres` superuser (which CNPG reserves). |
| **Backups** | `barmanObjectStore` → **per-env Cloudflare R2 bucket** with a **bucket-scoped token** (a leaked staging/dev key can't touch prod backups) + hourly `ScheduledBackup` + continuous WAL = PITR. Dev: none (disposable). |
| **Cutover** | **Blue/green** — new CNPG cluster runs alongside the old Postgres; repoint via a flag; old DB kept for rollback. **~2-min maintenance window for prod only** (rejected zero-downtime logical replication as over-engineering for a 14 MB DB). |
| **Web UI** | One shared **CloudBeaver** (Apache-2.0) for day-to-day data work; DR stays declarative (CNPG recovery manifests). |
| **DB naming** | Kept `raycesv3_production` on prod, but **renamed staging's DB to `raycesv3_staging`** — it had been confusingly `raycesv3_production` because staging runs `RAILS_ENV=production`. |

### Why deploy paths differ per env

- **dev & staging** are deployed by **Skaffold**, which reads values from
  `charts/rayces/values.remote.yaml` / `values.staging.yaml` in `rayces-v3`. The repoint
  flag (`externalDb.enabled`) lives there.
- **prod** is deployed by **Flux** via the HelmRelease in **this repo**
  (`apps/staging/rayces/helmrelease.yaml`) whose values are **inline** in `spec.values`. The
  repoint flag lives there.

Same chart, different value sources — a consequence of the two GitOps pipelines.

## 5. Implementation

### 5.1 The Helm chart change (guarded, default-inert; in `rayces-v3`)

A single `externalDb` block plus a `postgres.enabled` flag, both off by default:

- When `externalDb.enabled`: rails + sidekiq gain a **second `envFrom`** for the connection
  secret (its `POSTGRES_*` keys override the env secret), and the `wait-for-postgres` init
  container targets `externalDb.host:port` instead of the in-chart service.
- `postgres.enabled: false` drops the in-chart Postgres Deployment/Service/PVC/metrics, so an
  env can decommission the old DB after migrating.

With both flags at defaults, `helm template` output was **byte-identical to before** — the
change was inert for every environment until its values opted in.

### 5.2 The GitOps infra (this repo)

```
database/
  controllers/staging/          CNPG operator (Flux HelmRelease) + CloudBeaver
  instances/{dev,staging,prod}/ per-env: namespace, Cluster, app-secret(SOPS),
                                backup-secret(SOPS, staging/prod), ScheduledBackup
  app-secrets/{dev,staging,prod}/ rayces-db-connection secret (SOPS) → the app namespace
clusters/staging/database.yaml  Flux Kustomizations: database-controllers +
                                database-<env> + database-<env>-appsecret (each prune-isolated,
                                dependsOn controllers, SOPS decryption)
apps/staging/rayces/helmrelease.yaml   prod repoint (externalDb in spec.values) +
                                       reconcileStrategy: Revision
```

### 5.3 The secrets model

Per env, all SOPS-encrypted:
- `rayces-db-<env>-app` (basic-auth) → CNPG creates role + DB.
- `rayces-db-<env>-backup-s3` (R2 keys) → staging/prod only.
- `rayces-db-connection` (`POSTGRES_*`) → applied by Flux into the app namespace; the chart
  `envFrom`s it last so it wins. The same password is rendered into both the DB-side and
  app-side secrets (kept identical by hand — explicit, no extra operator).

## 6. The migration, environment by environment

### 6.1 Dev — prove the whole pattern at zero customer risk
Stood up the operator + CloudBeaver (one-time) and `rayces-db-dev`. Migrated with
`pg_dump -Fc | pg_restore` (streamed over **stdin** — CNPG pods have a read-only root
filesystem, so `kubectl cp` into `/tmp` fails). Repointed via `values.remote.yaml`, verified
(`server=16.4`, row counts intact), and **rehearsed rollback** (flip back to the old Postgres
→ `server=13.23`, then forward again). Decommissioned the old dev Postgres.

### 6.2 Staging — backups, PITR, and a prod-data dress rehearsal
Same slice **with** R2 backups. Two notable moves:
- **Seeded staging from a clone of live prod** (`pg_dump` prod → restore into staging CNPG) —
  giving staging a realistic dataset *and* rehearsing the exact prod dump/restore on real data.
- **Proved DR for real:** triggered a backup, then **recovered it into a throwaway scratch
  cluster** (`bootstrap.recovery` from R2) — 27 memberships came back. An untested backup is
  not a backup.

Renamed the DB to `raycesv3_staging`, decommissioned the old Postgres.

### 6.3 Prod — the scheduled maintenance window
Pre-staged `rayces-db-prod` (running, backed up) with zero impact. Then, in a ~2-min window:
`flux suspend` the HelmRelease → scale rails/sidekiq to 0 → final `pg_dump | pg_restore` →
flip `externalDb.enabled: true` in the HelmRelease values → `flux resume` + `reconcile`. Old
Postgres kept for a rollback soak.

## 7. Disaster recovery

Staging and prod archive WAL continuously and take hourly base backups to their own R2
bucket → **PITR** (RPO ≈ seconds). Recovery = bootstrap a *new* `Cluster` with
`bootstrap.recovery` pointing at the object store (optionally with a `recoveryTarget` time).
This same mechanism is how the database would be **lifted into a VPC**. The staging restore
was tested end-to-end into a scratch cluster.

## 8. War stories — five real gotchas

1. **CNPG read-only root filesystem.** `kubectl cp` of a dump into the pod fails; stream the
   archive over `kubectl exec -i` stdin to `pg_restore` instead.
2. **`pgcrypto` ownership.** Pre-creating it via `postInitApplicationSQL` runs as the
   *superuser*, so the app role can't `DROP/COMMENT` it during a `--clean` restore (2 harmless
   errors). For prod we left it out — the restore creates it as `raycesv3` (a *trusted*
   extension, DB-owner-creatable), error-free.
3. **`RAILS_ENV=production` on staging** made staging's DB literally named `raycesv3_production`
   — fixed by renaming to `raycesv3_staging` during the migration.
4. **Flux chart caching (the big one).** The prod HelmRelease defaulted to
   `reconcileStrategy: ChartVersion`: chart **template** changes never reached prod while
   `Chart.yaml` `version` stayed `0.2.0` — Flux kept rendering a cached old chart (the values
   flip applied, but old templates ignored it). The app stayed on the old DB despite
   `externalDb.enabled: true`. Fix: `spec.chart.spec.reconcileStrategy: Revision`.
5. **The `+` in the chart label.** `Revision` makes Flux append `+<commit>` to the chart
   version; the `helm.sh/chart` label used the raw version → `rayces-0.2.2+<sha>`, and `+` is
   **invalid in a k8s label**, failing every resource patch (and briefly taking prod down).
   Fix: sanitize with `… | replace "+" "_" | trunc 63 | trimSuffix "-"`. Recovery during the
   incident: `flux suspend` + manually scale the app back up on the in-sync old Postgres.

## 9. Tooling — CloudBeaver

A single CloudBeaver instance in `cnpg-system` provides a web UI to browse/query every CNPG
database (connections to each `rayces-db-<env>-rw` service). Access it with:

```bash
kubectl -n cnpg-system port-forward svc/cloudbeaver 8978:8978
# → http://localhost:8978  (complete first-run admin setup, then add a connection per env)
```

## 10. Outcome

| | Before | After |
|---|---|---|
| Engine | single `postgres:13` pod/env | CNPG-managed Postgres 16, per env |
| Backups | **none** | hourly base + continuous WAL → per-env R2 (staging/prod) |
| Recovery | impossible | **PITR, tested** |
| Secrets | mixed | **SOPS** everywhere, public-repo-safe |
| Blast radius | one action wipes data | isolated namespaces + Flux Kustomizations + scoped backup tokens |
| Portability | node-local only | object-store backups → restore into any cluster / VPC |

All three environments now run on CloudNativePG. The old Postgres pods are retained on prod
for a rollback soak and decommissioned via `postgres.enabled: false` once confidence is high.

## 11. Where things live

- **Chart:** `rayces-v3/charts/rayces` (the `externalDb` / `postgres.enabled` flags, sanitized
  labels).
- **Infra (this repo):** `database/**` + `clusters/staging/database.yaml`; prod repoint in
  `apps/staging/rayces/helmrelease.yaml`.
- **Design, plan & runbook:** in `rayces-v3` under `docs/superpowers/specs/`,
  `docs/superpowers/plans/`, and `docs/runbooks/rayces-db-cnpg.md`.
