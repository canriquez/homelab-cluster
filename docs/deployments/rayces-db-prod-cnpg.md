# Provision RayCES production PostgreSQL (CloudNativePG) + Flux wiring

**Branch:** `chore/rayces-db-prod-cnpg`
**Commit:** `2004771 configures db cnpg infra secrets and values`

## Summary

Adds the **production** RayCES database instance, managed by the CloudNativePG (CNPG)
operator, and introduces the Flux `Kustomization` entry point that reconciles the entire
`database/` stack (dev, staging, prod) into the cluster.

The production cluster (`rayces-db-prod`) runs PostgreSQL 16.4 with continuous WAL
archiving and hourly base backups to **Cloudflare R2**, giving point-in-time recovery
(PITR) with an effective RPO of seconds. All secrets are SOPS/Age-encrypted.

## Changes

| File | Type | Purpose |
|---|---|---|
| `clusters/staging/database.yaml` | new | Flux entry point — 7 `Kustomization` objects wiring `database/controllers` + dev/staging/prod instances & app-secrets |
| `database/instances/prod/namespace.yaml` | new | `rayces-db-prod` namespace |
| `database/instances/prod/cluster.yaml` | new | CNPG `Cluster` — PG 16.4, 10Gi storage, R2 backups, PITR |
| `database/instances/prod/scheduled-backup.yaml` | new | Hourly `ScheduledBackup` (top of every hour) |
| `database/instances/prod/app-secret.yaml` | new | DB owner credentials `rayces-db-prod-app` (SOPS-encrypted, `basic-auth`) |
| `database/instances/prod/backup-secret.yaml` | new | R2 S3 credentials `rayces-db-prod-backup-s3` (SOPS-encrypted) |
| `database/instances/prod/kustomization.yaml` | new | Kustomize resource list for the prod instance |
| `database/app-secrets/prod/connection-secret.yaml` | new | `rayces-db-connection` in `rayces-production` ns (SOPS-encrypted) |
| `database/app-secrets/prod/kustomization.yaml` | new | Kustomize resource list for the prod app-secret |

## Architecture

**Flux reconciliation order** (`clusters/staging/database.yaml`):

```
database-controllers   (./database/controllers/staging)   ← CNPG operator / CloudBeaver
        │ dependsOn
        ├── database-dev        (./database/instances/dev)
        ├── database-staging    (./database/instances/staging)
        └── database-prod       (./database/instances/prod)     ← this PR

database-{env}-appsecret  (./database/app-secrets/{env})        ← decoupled, no dependsOn
```

- Every instance `Kustomization` `dependsOn` `database-controllers` so the operator CRDs
  exist before any `Cluster` is applied.
- App-secret `Kustomization`s are reconciled independently — they project the connection
  secret into the consuming application namespace (`rayces-production`) rather than the DB
  namespace.
- All instance/app-secret Kustomizations enable SOPS decryption via the `sops-age` secret
  in `flux-system`.

**Production `Cluster` (`rayces-db-prod`):**

- **Engine:** `ghcr.io/cloudnative-pg/postgresql:16.4`, single instance, `unsupervised` primary updates.
- **Storage:** 10Gi on `local-path`.
- **Resources:** requests 256Mi / 100m, limits 1Gi / 1000m.
- **Bootstrap:** database `raycesv3_production`, owner `raycesv3`, credentials from `rayces-db-prod-app`.
- **Backups:** `barmanObjectStore` → `s3://rayces-db-prod-backups/` on Cloudflare R2,
  gzip-compressed WAL + data, **30-day retention**.
- **PITR:** continuous WAL archiving + hourly `ScheduledBackup` (`0 0 * * * *`, 6-field cron).

## Notable design decisions (vs. staging)

- **`pgcrypto` is intentionally not pre-created.** The data restore creates it as
  `raycesv3` (trusted extension + DB owner), keeping ownership correct and the restore
  error-free — unlike staging's pre-create approach.
- **No `CREATEDB` grant.** Production runs no test suite, so the app role follows least
  privilege.

## Secrets

All three new secrets are **SOPS/Age-encrypted** (`encrypted_regex: ^(data|stringData)$`),
verified encrypted in the diff:

- `rayces-db-prod-app` (`rayces-db-prod` ns) — type `kubernetes.io/basic-auth`, DB owner username/password.
- `rayces-db-prod-backup-s3` (`rayces-db-prod` ns) — `ACCESS_KEY_ID` / `ACCESS_SECRET_KEY` for R2.
- `rayces-db-connection` (`rayces-production` ns) — `POSTGRES_HOST/PORT/USER/PASSWORD/DB_PRODUCTION` for the app.

## Risk & rollout

- **Additive only** — 9 new files, 217 insertions, no modifications or deletions. No
  existing resources are touched.
- On merge to `main`, Flux reconciles within ~1 minute. The operator provisions the
  `rayces-db-prod` cluster; first base backup fires at the next top-of-hour.
- `prune: true` is set on all Kustomizations — removing these files later will tear down
  the corresponding cluster resources.

## Verification checklist

```bash
flux get kustomizations | grep database
kubectl get cluster -n rayces-db-prod
kubectl get pods -n rayces-db-prod
kubectl cnpg status rayces-db-prod -n rayces-db-prod   # if cnpg plugin installed
kubectl get scheduledbackup,backup -n rayces-db-prod
```

Confirm: cluster reports `Cluster in healthy state`, R2 receives WAL segments, and the
first hourly backup completes.
