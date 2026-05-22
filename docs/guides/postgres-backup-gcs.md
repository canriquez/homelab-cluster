# PostgreSQL Backup — GCS Implementation Guide

## Overview

This guide documents the complete implementation of an automated PostgreSQL backup pipeline for `rayces-production`, using `pg_dump` for logical backups, `rclone` for transport, and **Google Cloud Storage (GCS)** as the backup destination. The pipeline runs every 30 minutes, retains 14 days of backups, and includes a tested one-command Disaster Recovery procedure.

**Target environment:** `rayces-production` namespace, PostgreSQL 13, Flux CD GitOps-managed cluster.

---

## Full Backup vs Delta Backup — What You Need to Know

### What pg_dump does (always full)

`pg_dump` always produces a **complete logical snapshot** of the database. There is no incremental or delta mode. Every run dumps the entire database: all tables, all rows, all indexes, all constraints.

**Implications for 30-minute backups:**
- Each backup = full copy of the entire database at that moment
- 30-minute frequency × 14 days = **672 backup files** stored in GCS
- Cost depends entirely on database size

**Size and cost estimate for rayces-production:**

| Database size | Compressed dump | 672 files total | GCS cost/month |
|---|---|---|---|
| 100 MB | ~10 MB | ~6.7 GB | ~$0.13 |
| 500 MB | ~50 MB | ~33 GB | ~$0.66 |
| 2 GB | ~200 MB | ~130 GB | ~$2.60 |

GCS Standard Storage is $0.020/GB/month. GCS lifecycle policies automatically delete files older than 14 days — no manual pruning needed.

For a new app going live April 1st, database size will be small for months. 30-minute full dumps are the correct choice at this scale.

### Why not incremental/delta backups?

True delta backups for PostgreSQL require **WAL archiving** — PostgreSQL's Write-Ahead Log contains every change as it happens. Tools like WAL-G and CloudNativePG continuously archive WAL segments to object storage, enabling **Point-In-Time Recovery (PITR)** to any second in the past.

This requires:
- `archive_command` set in `postgresql.conf`
- Direct filesystem access to PostgreSQL data directory
- The postgres pod running as a StatefulSet with a stable identity

Your current setup (plain Deployment, no `postgresql.conf` customization) does not support WAL archiving without significant refactoring. **Phase 2 (post go-live)** covers the CloudNativePG migration path.

**For now: 30-minute pg_dump = max 30 minutes of data loss (RPO = 30 min). This is acceptable and production-grade for this stage.**

---

## Architecture

```
Every 30 minutes:
┌─────────────────────────────────────────────────────────┐
│  CronJob: postgres-backup (rayces-production)            │
│                                                          │
│  initContainer: pg-dump (postgres:13)                    │
│    pg_dump -h postgres-rayces -U $USER -d $DB            │
│    --format=custom → /backup/latest.dump                 │
│                                                          │
│  container: rclone-upload (rclone/rclone)                │
│    rclone copyto /backup/latest.dump                     │
│    → gcs:rayces-production-backups/backups/              │
│       backup-YYYYMMDD-HHMMSS.dump                        │
│                                                          │
│  emptyDir volume shared between containers               │
└─────────────────────────────────────────────────────────┘
         │
         ▼
GCS Bucket: rayces-production-backups
  └─ backups/
       ├─ backup-20260322-000000.dump
       ├─ backup-20260322-003000.dump
       ├─ ...
       └─ backup-YYYYMMDD-HHMMSS.dump
  Lifecycle policy: auto-delete after 14 days

         │
         ▼
PrometheusRule: alert if no successful backup in 2 hours
Grafana dashboard panel: backup history and last run time

         │  (on DR event)
         ▼
Restore Job: postgres-restore (applied manually)
  initContainer: rclone-download
    → downloads target backup from GCS to /restore/backup.dump
  container: pg-restore (postgres:13)
    → drops DB, recreates, pg_restore
```

**Why two containers instead of one custom image?**
- Uses only public, unmodified images (`postgres:13` and `rclone/rclone`)
- No custom Docker build required
- Each container does exactly one thing
- Easy to update independently

---

## Prerequisites

- GCP account (free tier sufficient for this data volume)
- `gcloud` CLI installed locally, authenticated
- `kubectl` and `flux` CLI connected to the cluster
- SOPS + Age key configured locally

---

## Implementation Steps

### Step 1 — Create GCS Bucket

Run locally (or via GCP Console):

```bash
# Set your GCP project ID
export PROJECT_ID=<your-gcp-project-id>
export BUCKET_NAME=rayces-production-backups

# Create the bucket (single region, standard storage)
gcloud storage buckets create gs://${BUCKET_NAME} \
  --project=${PROJECT_ID} \
  --location=US-CENTRAL1 \
  --uniform-bucket-level-access

# Verify
gcloud storage buckets describe gs://${BUCKET_NAME}
```

**In GCP Console (alternative):** Storage → Buckets → Create → name: `rayces-production-backups` → Region: single-region US-CENTRAL1 → Uniform access control.

---

### Step 2 — Create GCS Lifecycle Policy (14-day retention)

Create a lifecycle config file locally:

```bash
cat > /tmp/lifecycle.json << 'EOF'
{
  "rule": [
    {
      "action": {"type": "Delete"},
      "condition": {"age": 14}
    }
  ]
}
EOF

gcloud storage buckets update gs://rayces-production-backups \
  --lifecycle-file=/tmp/lifecycle.json
```

Verify in Console: Storage → `rayces-production-backups` → Lifecycle → should show "Delete after 14 days".

---

### Step 3 — Create GCP Service Account and Key

```bash
export PROJECT_ID=<your-gcp-project-id>
export SA_NAME=rayces-postgres-backup

# Create service account
gcloud iam service-accounts create ${SA_NAME} \
  --display-name="Rayces PostgreSQL Backup" \
  --project=${PROJECT_ID}

# Grant Storage Object Admin on the bucket only (not project-wide)
gcloud storage buckets add-iam-policy-binding gs://rayces-production-backups \
  --member="serviceAccount:${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"

# Create and download the JSON key
gcloud iam service-accounts keys create /tmp/rayces-postgres-backup-sa-key.json \
  --iam-account="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

# Confirm the file was created
cat /tmp/rayces-postgres-backup-sa-key.json | python3 -m json.tool | head -5
```

**Important:** The key file lives only at `/tmp/rayces-postgres-backup-sa-key.json`. It will be SOPS-encrypted before committing. Delete it locally after encryption.

---

### Step 4 — Create the Kubernetes Secret (SOPS-encrypted)

Create the secret template:

```bash
# One-line the JSON key (required for stringData embedding)
SA_KEY=$(cat /tmp/rayces-postgres-backup-sa-key.json | tr -d '\n')

cat > apps/staging/rayces/secrets/postgres-backup-gcs-secret.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: postgres-backup-gcs-credentials
  namespace: rayces-production
type: Opaque
stringData:
  service-account.json: |
$(cat /tmp/rayces-postgres-backup-sa-key.json | sed 's/^/    /')
EOF
```

Encrypt it:

```bash
sops --encrypt --in-place \
  apps/staging/rayces/secrets/postgres-backup-gcs-secret.yaml
```

Delete the local key file:

```bash
rm /tmp/rayces-postgres-backup-sa-key.json
```

---

### Step 5 — Create the Backup CronJob

Create `apps/staging/rayces/postgres-backup-cronjob.yaml`:

```yaml
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: rayces-production
spec:
  schedule: "*/30 * * * *"        # every 30 minutes
  concurrencyPolicy: Forbid        # skip if previous job still running
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  jobTemplate:
    spec:
      backoffLimit: 2              # retry twice before marking failed
      template:
        spec:
          restartPolicy: OnFailure
          volumes:
            - name: backup-volume
              emptyDir: {}
            - name: gcs-credentials
              secret:
                secretName: postgres-backup-gcs-credentials
          initContainers:
            - name: pg-dump
              image: postgres:13
              command: ["/bin/bash", "-c"]
              args:
                - |
                  set -euo pipefail
                  echo "[backup] Starting pg_dump at $(date +%Y-%m-%dT%H:%M:%SZ)"
                  PGPASSWORD="${POSTGRES_PASSWORD}" pg_dump \
                    -h "${POSTGRES_HOST}" \
                    -U "${POSTGRES_USER}" \
                    -d "${POSTGRES_DB_PRODUCTION}" \
                    --format=custom \
                    --no-password \
                    -f /backup/latest.dump
                  echo "[backup] pg_dump complete. Size: $(du -sh /backup/latest.dump | cut -f1)"
              env:
                - name: POSTGRES_USER
                  valueFrom:
                    secretKeyRef:
                      name: raycesv3-environment
                      key: POSTGRES_USER
                - name: POSTGRES_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: raycesv3-environment
                      key: POSTGRES_PASSWORD
                - name: POSTGRES_DB_PRODUCTION
                  valueFrom:
                    secretKeyRef:
                      name: raycesv3-environment
                      key: POSTGRES_DB_PRODUCTION
                - name: POSTGRES_HOST
                  valueFrom:
                    secretKeyRef:
                      name: raycesv3-environment
                      key: POSTGRES_HOST
              volumeMounts:
                - name: backup-volume
                  mountPath: /backup
          containers:
            - name: rclone-upload
              image: rclone/rclone:1.68
              command: ["/bin/sh", "-c"]
              args:
                - |
                  set -euo pipefail
                  TIMESTAMP=$(date +%Y%m%d-%H%M%S)
                  DEST="gcs:${GCS_BUCKET}/backups/backup-${TIMESTAMP}.dump"
                  echo "[upload] Uploading to ${DEST}"
                  rclone copyto /backup/latest.dump "${DEST}" \
                    --gcs-service-account-file /gcs-credentials/service-account.json \
                    --gcs-bucket-policy-only
                  echo "[upload] Upload complete: backup-${TIMESTAMP}.dump"
              env:
                - name: GCS_BUCKET
                  value: "rayces-production-backups"
                - name: RCLONE_CONFIG_GCS_TYPE
                  value: "google cloud storage"
              volumeMounts:
                - name: backup-volume
                  mountPath: /backup
                - name: gcs-credentials
                  mountPath: /gcs-credentials
                  readOnly: true
```

---

### Step 6 — Create Restore Job Template

Create `apps/staging/rayces/postgres-restore-job.yaml`.

**This file is NOT added to kustomization.yaml.** It is a template applied manually during DR. Keep it committed to the repo so it is always available.

```yaml
---
# RESTORE JOB — Apply manually during disaster recovery.
# Do NOT add to kustomization.yaml.
#
# Usage:
#   1. Set BACKUP_FILE to the target backup (or leave as "latest" to auto-select most recent)
#   2. kubectl apply -f apps/staging/rayces/postgres-restore-job.yaml
#   3. Monitor: kubectl logs -f -n rayces-production job/postgres-restore -c pg-restore
#   4. After completion: kubectl delete -f apps/staging/rayces/postgres-restore-job.yaml
#
# To restore the latest backup automatically, set BACKUP_FILE to "latest".
# To restore a specific backup, set BACKUP_FILE to the exact filename:
#   e.g. "backup-20260322-120000.dump"
#
apiVersion: batch/v1
kind: Job
metadata:
  name: postgres-restore
  namespace: rayces-production
spec:
  backoffLimit: 0                   # no retries — restore is destructive
  template:
    spec:
      restartPolicy: Never
      volumes:
        - name: restore-volume
          emptyDir: {}
        - name: gcs-credentials
          secret:
            secretName: postgres-backup-gcs-credentials
      initContainers:
        - name: rclone-download
          image: rclone/rclone:1.68
          command: ["/bin/sh", "-c"]
          args:
            - |
              set -euo pipefail
              if [ "${BACKUP_FILE}" = "latest" ]; then
                echo "[restore] Finding latest backup in GCS..."
                BACKUP_FILE=$(rclone lsf "gcs:${GCS_BUCKET}/backups/" \
                  --gcs-service-account-file /gcs-credentials/service-account.json \
                  --gcs-bucket-policy-only \
                  | grep "^backup-" | sort | tail -1 | tr -d '\n')
                echo "[restore] Latest backup: ${BACKUP_FILE}"
              fi
              SRC="gcs:${GCS_BUCKET}/backups/${BACKUP_FILE}"
              echo "[restore] Downloading ${SRC}..."
              rclone copyto "${SRC}" /restore/backup.dump \
                --gcs-service-account-file /gcs-credentials/service-account.json \
                --gcs-bucket-policy-only
              echo "[restore] Download complete. Size: $(du -sh /restore/backup.dump | cut -f1)"
          env:
            - name: GCS_BUCKET
              value: "rayces-production-backups"
            - name: BACKUP_FILE
              value: "latest"          # CHANGE THIS to a specific filename if needed
            - name: RCLONE_CONFIG_GCS_TYPE
              value: "google cloud storage"
          volumeMounts:
            - name: restore-volume
              mountPath: /restore
            - name: gcs-credentials
              mountPath: /gcs-credentials
              readOnly: true
      containers:
        - name: pg-restore
          image: postgres:13
          command: ["/bin/bash", "-c"]
          args:
            - |
              set -euo pipefail
              echo "[restore] ========================================"
              echo "[restore] STARTING DATABASE RESTORE"
              echo "[restore] Target DB: ${POSTGRES_DB_PRODUCTION}"
              echo "[restore] Host:      ${POSTGRES_HOST}"
              echo "[restore] ========================================"

              # Terminate all existing connections to the target database
              echo "[restore] Terminating existing connections..."
              PGPASSWORD="${POSTGRES_PASSWORD}" psql \
                -h "${POSTGRES_HOST}" \
                -U "${POSTGRES_USER}" \
                -d postgres \
                -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = '${POSTGRES_DB_PRODUCTION}' AND pid <> pg_backend_pid();"

              # Drop and recreate the database
              echo "[restore] Dropping database ${POSTGRES_DB_PRODUCTION}..."
              PGPASSWORD="${POSTGRES_PASSWORD}" psql \
                -h "${POSTGRES_HOST}" \
                -U "${POSTGRES_USER}" \
                -d postgres \
                -c "DROP DATABASE IF EXISTS \"${POSTGRES_DB_PRODUCTION}\";"

              echo "[restore] Creating database ${POSTGRES_DB_PRODUCTION}..."
              PGPASSWORD="${POSTGRES_PASSWORD}" psql \
                -h "${POSTGRES_HOST}" \
                -U "${POSTGRES_USER}" \
                -d postgres \
                -c "CREATE DATABASE \"${POSTGRES_DB_PRODUCTION}\";"

              # Restore from dump
              echo "[restore] Running pg_restore..."
              PGPASSWORD="${POSTGRES_PASSWORD}" pg_restore \
                -h "${POSTGRES_HOST}" \
                -U "${POSTGRES_USER}" \
                -d "${POSTGRES_DB_PRODUCTION}" \
                --no-owner \
                --no-acl \
                --exit-on-error \
                /restore/backup.dump

              echo "[restore] ========================================"
              echo "[restore] RESTORE COMPLETE"
              echo "[restore] ========================================"
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: raycesv3-environment
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: raycesv3-environment
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB_PRODUCTION
              valueFrom:
                secretKeyRef:
                  name: raycesv3-environment
                  key: POSTGRES_DB_PRODUCTION
            - name: POSTGRES_HOST
              valueFrom:
                secretKeyRef:
                  name: raycesv3-environment
                  key: POSTGRES_HOST
          volumeMounts:
            - name: restore-volume
              mountPath: /restore
```

---

### Step 7 — Create PrometheusRule Alert

Create `monitoring/configs/staging/postgres-backup-alert.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: postgres-backup-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: postgres-backup
      interval: 1m
      rules:
        - alert: PostgresBackupMissed
          expr: |
            (time() - kube_cronjob_status_last_successful_time{
              namespace="rayces-production",
              cronjob="postgres-backup"
            }) > 7200
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "PostgreSQL backup has not run successfully in over 2 hours"
            description: >
              The postgres-backup CronJob in rayces-production has not completed
              successfully in the last 2 hours ({{ $value | humanizeDuration }} ago).
              Check: kubectl get jobs -n rayces-production
              Logs: kubectl logs -n rayces-production -l job-name=... -c rclone-upload
        - alert: PostgresBackupJobFailed
          expr: |
            kube_job_status_failed{
              namespace="rayces-production",
              job_name=~"postgres-backup-.*"
            } > 0
          for: 1m
          labels:
            severity: warning
          annotations:
            summary: "PostgreSQL backup job failed"
            description: "Job {{ $labels.job_name }} in rayces-production has failed."
```

Add to `monitoring/configs/staging/kustomization.yaml`:

```yaml
resources:
  - ...existing entries...
  - postgres-backup-alert.yaml
```

---

### Step 8 — Update kustomization.yaml

Edit `apps/staging/rayces/kustomization.yaml` to add the new files:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - gitrepository.yaml
  - helmrelease.yaml
  - image-automation.yaml
  - postgres-backup-cronjob.yaml          # new
  - secrets/environment-secret.yaml
  - secrets/ghcr-secret.yaml
  - secrets/tunnel-secret.yaml
  - secrets/github-secret.yaml
  - secrets/image-automation-git-secret.yaml
  - secrets/ghcr-flux-system-secret.yaml
  - secrets/postgres-backup-gcs-secret.yaml  # new
```

**Note:** `postgres-restore-job.yaml` is NOT added here — it is applied manually.

---

### Step 9 — Verify Backup Pipeline

After committing and pushing:

```bash
# 1. Check Flux reconciled the new resources
flux get kustomizations
kubectl get cronjob postgres-backup -n rayces-production

# 2. Trigger a manual backup immediately (don't wait 30 min)
kubectl create job --from=cronjob/postgres-backup manual-backup-test \
  -n rayces-production

# 3. Watch the job
kubectl get jobs -n rayces-production -w

# 4. Watch the logs (both containers)
# pg_dump phase:
kubectl logs -n rayces-production \
  -l job-name=manual-backup-test -c pg-dump --follow

# rclone upload phase (after pg_dump completes):
kubectl logs -n rayces-production \
  -l job-name=manual-backup-test -c rclone-upload --follow

# 5. Verify file exists in GCS
gcloud storage ls gs://rayces-production-backups/backups/

# 6. Clean up test job
kubectl delete job manual-backup-test -n rayces-production
```

Expected output from rclone-upload:
```
[upload] Uploading to gcs:rayces-production-backups/backups/backup-20260322-190000.dump
[upload] Upload complete: backup-20260322-190000.dump
```

---

## Disaster Recovery Procedure

### When to trigger a restore

Trigger a full restore when:
- The PostgreSQL data volume is corrupted or lost
- A destructive migration or operation damaged the data and cannot be rolled back
- The postgres pod cannot start due to data directory issues

**Do NOT restore for:**
- Application bugs (fix at application layer)
- Single-row data issues (fix with SQL)
- Pod crashes that self-heal (Kubernetes restarts the pod automatically)

---

### DR Step 1 — Scale down the application

Before restoring, stop all application writes to prevent data corruption during restore:

```bash
kubectl scale deployment rails-rayces --replicas=0 -n rayces-production
kubectl scale deployment sidekiq-rayces --replicas=0 -n rayces-production
kubectl scale deployment nextjs-rayces --replicas=0 -n rayces-production
```

---

### DR Step 2 — Select the backup to restore

List available backups (newest last):

```bash
gcloud storage ls gs://rayces-production-backups/backups/ | sort
```

Or via rclone:

```bash
kubectl run rclone-list --rm -it --restart=Never \
  --image=rclone/rclone:1.68 \
  --overrides='{"spec":{"volumes":[{"name":"gcs","secret":{"secretName":"postgres-backup-gcs-credentials"}}],"containers":[{"name":"rclone-list","image":"rclone/rclone:1.68","command":["/bin/sh","-c"],"args":["rclone lsf gcs:rayces-production-backups/backups/ --gcs-service-account-file /gcs/service-account.json --gcs-bucket-policy-only | sort"],"volumeMounts":[{"name":"gcs","mountPath":"/gcs"}]}]}}' \
  -n rayces-production
```

Choose the backup you want. Options:
- **Latest backup** — set `BACKUP_FILE: "latest"` in the restore job (auto-selects newest)
- **Specific backup** — set `BACKUP_FILE: "backup-20260322-120000.dump"`

---

### DR Step 3 — Apply the Restore Job

If restoring the latest backup (default):

```bash
kubectl apply -f apps/staging/rayces/postgres-restore-job.yaml
```

If restoring a specific backup, first edit the file:

```bash
# Edit the BACKUP_FILE value in the initContainer env section
# Change: value: "latest"
# To:     value: "backup-20260322-120000.dump"
kubectl apply -f apps/staging/rayces/postgres-restore-job.yaml
```

---

### DR Step 4 — Monitor the restore

```bash
# Watch job progress
kubectl get job postgres-restore -n rayces-production -w

# Download phase logs
kubectl logs -f -n rayces-production \
  -l job-name=postgres-restore -c rclone-download

# Restore phase logs
kubectl logs -f -n rayces-production \
  -l job-name=postgres-restore -c pg-restore
```

Successful restore logs look like:

```
[restore] ========================================
[restore] STARTING DATABASE RESTORE
[restore] Target DB: rayces_production
[restore] Host:      postgres-rayces
[restore] ========================================
[restore] Terminating existing connections...
[restore] Dropping database rayces_production...
[restore] Creating database rayces_production...
[restore] Running pg_restore...
[restore] ========================================
[restore] RESTORE COMPLETE
[restore] ========================================
```

---

### DR Step 5 — Scale the application back up

```bash
kubectl scale deployment rails-rayces --replicas=2 -n rayces-production
kubectl scale deployment sidekiq-rayces --replicas=1 -n rayces-production
kubectl scale deployment nextjs-rayces --replicas=2 -n rayces-production
```

---

### DR Step 6 — Verify and clean up

```bash
# Verify application health
kubectl get pods -n rayces-production
kubectl logs -n rayces-production deploy/rails-rayces --tail=20

# Delete the restore job (required before running again)
kubectl delete -f apps/staging/rayces/postgres-restore-job.yaml
```

---

## Testing the Restore (Non-Destructive)

Test the restore procedure without touching production data by restoring to a **separate test database** on the same postgres instance.

### Test procedure

**1. Trigger a backup and note the filename:**

```bash
kubectl create job --from=cronjob/postgres-backup restore-test-backup \
  -n rayces-production
# Wait for completion, then:
gcloud storage ls gs://rayces-production-backups/backups/ | sort | tail -3
# Note the latest filename, e.g. backup-20260322-190000.dump
```

**2. Create a test restore job** that restores to `rayces_restore_test` instead of the production database:

```bash
# Create a one-off restore job targeting a test DB
kubectl run restore-test --rm -it --restart=Never \
  --image=postgres:13 \
  --env="POSTGRES_HOST=postgres-rayces" \
  --env="PGPASSWORD=$(kubectl get secret raycesv3-environment -n rayces-production -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d)" \
  --env="POSTGRES_USER=$(kubectl get secret raycesv3-environment -n rayces-production -o jsonpath='{.data.POSTGRES_USER}' | base64 -d)" \
  -n rayces-production \
  -- /bin/bash -c "psql -h postgres-rayces -U \$POSTGRES_USER -d postgres -c 'CREATE DATABASE rayces_restore_test;'"
```

**3. Download the backup and restore to test DB:**

Apply a modified restore job pointing to `rayces_restore_test`:

```bash
# Quick one-liner restore to test DB (downloads backup from GCS + restores)
# Step A: download backup locally from GCS
gcloud storage cp \
  gs://rayces-production-backups/backups/backup-20260322-190000.dump \
  /tmp/test-restore.dump

# Step B: port-forward to postgres
kubectl port-forward svc/postgres-rayces 5433:5432 -n rayces-production &

# Step C: restore to test DB
PGPASSWORD=<POSTGRES_PASSWORD> pg_restore \
  -h localhost -p 5433 \
  -U <POSTGRES_USER> \
  -d rayces_restore_test \
  --no-owner --no-acl \
  /tmp/test-restore.dump

# Step D: verify row counts match production
PGPASSWORD=<POSTGRES_PASSWORD> psql \
  -h localhost -p 5433 \
  -U <POSTGRES_USER> \
  -d rayces_restore_test \
  -c "SELECT schemaname, tablename, n_live_tup FROM pg_stat_user_tables ORDER BY n_live_tup DESC;"

# Step E: clean up test database
PGPASSWORD=<POSTGRES_PASSWORD> psql \
  -h localhost -p 5433 \
  -U <POSTGRES_USER> \
  -d postgres \
  -c "DROP DATABASE rayces_restore_test;"

kill %1  # stop port-forward
rm /tmp/test-restore.dump
```

**A successful test confirms:**
- Backup file is valid and not corrupted
- pg_restore can read it without errors
- Row counts match expected data
- The full DR procedure will work when needed

---

## Monitoring

### Check backup history

```bash
# List all backup jobs (last 10)
kubectl get jobs -n rayces-production | grep postgres-backup | tail -10

# Check CronJob last schedule and status
kubectl describe cronjob postgres-backup -n rayces-production | tail -15
```

### Check last successful backup time

```bash
# Via kube-state-metrics (Prometheus)
# Query in Grafana: kube_cronjob_status_last_successful_time{cronjob="postgres-backup"}
```

### List backups in GCS

```bash
gcloud storage ls gs://rayces-production-backups/backups/ | sort | tail -10
```

### Check total storage used

```bash
gcloud storage du gs://rayces-production-backups/ --summarize
```

---

## File Inventory

| File | Purpose |
|---|---|
| `apps/staging/rayces/postgres-backup-cronjob.yaml` | CronJob — runs every 30 min |
| `apps/staging/rayces/postgres-restore-job.yaml` | Restore Job template — apply manually for DR |
| `apps/staging/rayces/secrets/postgres-backup-gcs-secret.yaml` | GCS service account key (SOPS-encrypted) |
| `apps/staging/rayces/kustomization.yaml` | Updated to include CronJob and secret |
| `monitoring/configs/staging/postgres-backup-alert.yaml` | PrometheusRule — alerts on backup failure |
| `monitoring/configs/staging/kustomization.yaml` | Updated to include alert rule |

---

## Phase 2 — WAL Archiving (Post Go-Live Roadmap)

After the application is stable in production, the recommended upgrade path is:

1. **Migrate postgres Deployment → CloudNativePG Cluster** — operator-managed PostgreSQL with built-in HA, WAL archiving, and PITR
2. **Configure WAL archiving to GCS** — continuous WAL segments uploaded as they fill (~every few minutes)
3. **PITR** — restore to any point in time, not just every 30 minutes
4. **Automatic failover** — CloudNativePG promotes a replica if primary fails

This gives RPO < 5 minutes and RTO < 2 minutes. The GCS bucket and service account created in this guide can be reused in Phase 2.

---

## Quick Reference Card

```bash
# Trigger manual backup now
kubectl create job --from=cronjob/postgres-backup manual-$(date +%H%M) -n rayces-production

# List backups in GCS
gcloud storage ls gs://rayces-production-backups/backups/ | sort | tail -5

# Trigger restore (latest backup)
kubectl apply -f apps/staging/rayces/postgres-restore-job.yaml

# Watch restore
kubectl logs -f -n rayces-production -l job-name=postgres-restore -c pg-restore

# Clean up restore job
kubectl delete -f apps/staging/rayces/postgres-restore-job.yaml

# Check backup alert
kubectl get prometheusrule postgres-backup-alerts -n monitoring
```
