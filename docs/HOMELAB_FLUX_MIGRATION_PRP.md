# PRP: Migrate `rayces-prd` to Flux-Managed `rayces-production`

**Document version:** 1.0
**Date:** 2025-03-22
**Author:** Carlos Anriquez
**Target repo:** `homelab-cluster` (https://github.com/canriquez/homelab-cluster)
**Source repo reference:** `rayces-v3` (https://github.com/canriquez/rayces-v3)

---

## Purpose

This document is an implementation guide for a Claude Code agent working inside the
`homelab-cluster` GitOps repo. It describes every file to create or modify to bring the
`rayces-production` namespace under Flux CD management — following the exact same pattern
already used by the `churlesflows` application on this cluster.

The goal is a **parallel blue/green migration**:

| | Old (`rayces-prd`) | New (`rayces-production`) |
|---|---|---|
| Managed by | Manual `skaffold run` + shell script | Flux CD (this repo) |
| URL | `flows.rayces.com` | `citas.rayces.com` |
| Secrets | Plaintext `.env.prd` applied by shell script | SOPS-encrypted, committed here |
| Image updates | Manual re-run | Update image tag, commit, Flux deploys |
| Monitoring | None | `postgres_exporter` sidecar + ServiceMonitor |

`rayces-prd` stays untouched and live throughout. Only when `rayces-production` is
fully validated do we cut over and decommission `rayces-prd`.

---

## Agent Instructions

> **Read this section before doing anything else.**

1. **Confirm before acting.** Before writing any file, read the relevant existing files in
   this repo and compare them to what is described here. If anything conflicts with what
   you observe in the repo (different paths, different naming conventions, different SOPS
   config), adjust the plan and confirm with the user before proceeding.

2. **Follow the churlesflows pattern exactly.** When in doubt about structure, read
   `apps/staging/churlesflows/` as the canonical reference. Match file naming, indentation,
   and SOPS encryption style to what is already there.

3. **Never commit or push.** Only create/edit files. The user handles all git operations.

4. **One step at a time.** After completing each numbered step below, stop and confirm
   the output with the user before moving to the next step.

5. **Verify cluster state first.** Before starting Phase 1, run the verification commands
   in the "Pre-flight checks" section to confirm the live state matches what this document
   expects.

6. **Secrets require user action.** You cannot create SOPS-encrypted secrets on your own —
   you will produce the plaintext YAML template and the user will encrypt and verify it.
   Flag every secret file clearly.

---

## Cluster Facts (confirmed as of 2025-03-22)

These were verified via `kubectl` against the live cluster. Confirm they are still accurate
before starting.

```
Cluster context:           remote-k3s-ubuntu
Flux GitRepository:        ssh://git@github.com/canriquez/homelab-cluster (main branch)
Flux apps Kustomization:   path: ./apps/staging   (prune: true, sops decryption enabled)
Flux monitoring-configs:   path: ./monitoring/configs/staging

Namespaces relevant to this work:
  monitoring         — kube-prometheus-stack, Prometheus, Grafana (195d old, stable)
  churlesflows       — n8n, PostgreSQL, Redis (195d old, Flux-managed reference)
  rayces-prd         — current manual production (2d old, NOT Flux-managed)

Prometheus serviceMonitorSelector label:  release: kube-prometheus-stack
SOPS Age public key:  age1c8uz8qzz095uspf99rfpketpcl9fk3zpwtjkts0nx4tyxctjqdyspqzj36
SOPS config location: clusters/staging/.sops.yaml

Current rayces-prd images (baseline for tag reference):
  Backend:   ghcr.io/canriquez/rayces-backend:f5e7ad7
  Frontend:  ghcr.io/canriquez/rayces-frontend:f5e7ad7
  Postgres:  postgres:13
  Redis:     redis:7-alpine
  Cloudflare: cloudflare/cloudflared:2024.12.0

Secret name in rayces-prd:  raycesv3-environment
Secret keys (61 total, listed in Prerequisites section)

Flux Image Automation controllers: NOT installed on this cluster
  → Image tag updates will be done manually (commit new tag value) for now
  → Adding Image Automation is a separate future initiative
```

---

## Pre-flight Checks

Run these commands first and confirm the output matches expectations before writing any file.

```bash
# 1. Confirm Flux kustomizations and their paths
flux get kustomizations

# 2. Confirm monitoring namespace has Prometheus with correct label selector
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}'
# Expected: {"matchLabels":{"release":"kube-prometheus-stack"}}

# 3. Confirm rayces-prd is running and healthy (will stay untouched)
kubectl get pods -n rayces-prd

# 4. Confirm rayces-production namespace does NOT exist yet
kubectl get namespace rayces-production
# Expected: Error from server (NotFound)

# 5. Confirm churlesflows reference pattern (read this as your style guide)
ls apps/staging/churlesflows/

# 6. Confirm SOPS config exists and has the Age key
cat clusters/staging/.sops.yaml

# 7. Check if a GitRepository for rayces-v3 already exists
kubectl get gitrepository -n flux-system
# Expected: only flux-system listed (no rayces-v3 yet)
```

---

## Prerequisites (User Actions — Done Before Agent Starts)

The following steps **require the user** to run CLI commands outside the repo. Complete
all of them before asking the agent to begin Phase 1.

### P1 — Create the new Cloudflare tunnel

On your local machine (where `cloudflared` is installed and authenticated):

```bash
# Create the new tunnel
cloudflared tunnel create rayces-production

# Route the new domain to it (citas.rayces.com must be in your Cloudflare account)
cloudflared tunnel route dns rayces-production citas.rayces.com

# Note the tunnel UUID that was created
cloudflared tunnel list | grep rayces-production
# Save the UUID — you will need it for the SOPS secret below

# Export the credentials JSON path (you will encrypt this file in P3)
ls ~/.cloudflared/   # find the file named <UUID>.json
```

### P2 — Gather the production secret values

The new namespace needs a clone of the `raycesv3-environment` secret, with values
appropriate for `rayces-production`. Decode the current values from `rayces-prd`:

```bash
kubectl get secret raycesv3-environment -n rayces-prd \
  -o go-template='{{range $k,$v := .data}}{{$k}}={{$v | base64decode}}{{"\n"}}{{end}}' \
  > /tmp/rayces-production.env
```

Review `/tmp/rayces-production.env` and update the following values for the new namespace:

| Key | Old value (rayces-prd) | New value (rayces-production) |
|---|---|---|
| `APP_HOST` | `flows.rayces.com` | `citas.rayces.com` |
| `FRONTEND_URL` | `https://flows.rayces.com` | `https://citas.rayces.com` |
| `NEXTAUTH_URL` | `https://flows.rayces.com` | `https://citas.rayces.com` |
| `NEXT_PUBLIC_DOMAIN` | `flows.rayces.com` | `citas.rayces.com` |
| `NEXT_PUBLIC_API_URL` | (update if domain-specific) | update to `citas.rayces.com` |
| `NEXT_PUBLIC_RAILS_BACKEND` | (update if domain-specific) | update to `citas.rayces.com` |
| `RACK_ENV` | `production` | `production` (unchanged) |
| `RAILS_ENV` | `production` | `production` (unchanged) |

All other keys can be copied verbatim from `rayces-prd`.

### P3 — Create and encrypt the SOPS secret files

You will need three secret files encrypted with SOPS. The agent will create the plaintext
YAML templates in `apps/staging/rayces/secrets/`; you then encrypt them.

Encrypt each file after the agent creates it:
```bash
sops --encrypt --in-place apps/staging/rayces/secrets/environment-secret.yaml
sops --encrypt --in-place apps/staging/rayces/secrets/ghcr-secret.yaml
sops --encrypt --in-place apps/staging/rayces/secrets/tunnel-secret.yaml
```

### P4 — Confirm GitHub access for Flux GitRepository

Flux needs to pull the `rayces-v3` chart from GitHub. Check whether the existing Flux
SSH deploy key has read access to `canriquez/rayces-v3`:

```bash
# Check current deploy key
kubectl get secret flux-system -n flux-system -o jsonpath='{.data.identity}' | \
  base64 -d | ssh-keygen -l -f -
```

If the key does NOT have access to `rayces-v3`, add it as a read-only deploy key in
GitHub → `rayces-v3` → Settings → Deploy Keys.

Alternatively, the agent will create the GitRepository with HTTPS + a GitHub PAT stored
as a secret (simpler for a private repo). You will need a GitHub PAT with `read:packages`
and `contents:read` scope.

---

## Phase 1 — Repo Structure and GitRepository Source

### Step 1.1 — Create the directory structure

The agent should create these empty directories (by creating placeholder/real files in them):

```
apps/
└── staging/
    └── rayces/           ← mirrors apps/staging/churlesflows/ structure
        ├── namespace.yaml
        ├── kustomization.yaml
        ├── gitrepository.yaml
        ├── helmrelease.yaml
        └── secrets/
            ├── environment-secret.yaml   ← PLAINTEXT TEMPLATE (user encrypts)
            ├── ghcr-secret.yaml          ← PLAINTEXT TEMPLATE (user encrypts)
            └── tunnel-secret.yaml        ← PLAINTEXT TEMPLATE (user encrypts)
```

Also add one file to monitoring:
```
monitoring/
└── configs/
    └── staging/
        └── rayces-postgres-servicemonitor.yaml   ← NEW
```

### Step 1.2 — `apps/staging/rayces/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: rayces-production
```

### Step 1.3 — `apps/staging/rayces/kustomization.yaml`

This is the Flux Kustomization resource (not a kustomize file — it is a Flux CRD).
Check the churlesflows equivalent for the exact format used in this repo.

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: rayces-production
  namespace: flux-system
spec:
  interval: 1m
  path: ./apps/staging/rayces
  prune: true
  retryInterval: 1m
  sourceRef:
    kind: GitRepository
    name: flux-system
  decryption:
    provider: sops
    secretRef:
      name: sops-age
  dependsOn:
    - name: flux-system
```

> **Agent note:** Check `apps/staging/churlesflows/kustomization.yaml` (if it exists as
> a Kustomization CRD) or check how churlesflows is registered in `clusters/staging/apps.yaml`.
> The churlesflows apps may be listed directly in apps.yaml rather than having their own
> Kustomization CRD. Match whatever pattern churlesflows uses.

### Step 1.4 — Register in `clusters/staging/apps.yaml`

Open `clusters/staging/apps.yaml` and add the rayces entry following the same format as
the churlesflows entry. Example (adjust to match actual file format):

```yaml
# If apps.yaml uses a Kustomization that recurses into apps/staging/:
# No change needed — Flux will pick up the new directory automatically.

# If apps.yaml lists each app explicitly, add:
- path: ./apps/staging/rayces
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

> **Agent:** Read `clusters/staging/apps.yaml` first to determine which pattern applies,
> then make the minimal correct change.

### Step 1.5 — `apps/staging/rayces/gitrepository.yaml`

This Flux source tells Flux where to pull the Helm chart from.

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: rayces-v3
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/canriquez/rayces-v3
  ref:
    branch: main
  secretRef:
    name: github-rayces-credentials   # created in Step 1.6
```

### Step 1.6 — `apps/staging/rayces/secrets/github-secret.yaml` (PLAINTEXT TEMPLATE)

> **STOP — User must encrypt this file before committing.**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-rayces-credentials
  namespace: flux-system
type: Opaque
stringData:
  username: canriquez
  password: REPLACE_WITH_GITHUB_PAT   # GitHub PAT with contents:read scope
```

After the agent creates this file, encrypt it:
```bash
sops --encrypt --in-place apps/staging/rayces/secrets/github-secret.yaml
```

---

## Phase 2 — HelmRelease

### Step 2.1 — `apps/staging/rayces/helmrelease.yaml`

This deploys the `rayces` Helm chart from the `rayces-v3` repo into the
`rayces-production` namespace.

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: rayces
  namespace: rayces-production
spec:
  interval: 5m
  chart:
    spec:
      chart: charts/rayces
      sourceRef:
        kind: GitRepository
        name: rayces-v3
        namespace: flux-system
      interval: 1m
  # Override values inline (equivalent to values.production.yaml but for this namespace)
  values:
    global:
      namespace: rayces-production
      secretName: raycesv3-environment

    images:
      backend:
        repository: ghcr.io/canriquez/rayces-backend
        tag: "f5e7ad7"          # ← Update this manually on each deploy
        pullPolicy: Always
        pullSecret: ghcr-credentials
      frontend:
        repository: ghcr.io/canriquez/rayces-frontend
        tag: "f5e7ad7"          # ← Update this manually on each deploy
        pullPolicy: Always
        pullSecret: ghcr-credentials

    rails:
      replicas: 2
      env: production
      autoSeed: false

    sidekiq:
      concurrency: "10"

    nextjs:
      replicas: 2
      command: ["node", "server.js"]
      nodeEnv: production

    mailcatcher:
      enabled: false

    cloudflare:
      enabled: true
      replicas: 2
      tunnel: rayces-production       # NEW tunnel name
      ingress:
        appHostname: citas.rayces.com  # NEW domain

    postgres:
      metrics:
        enabled: true                  # Enables postgres_exporter sidecar
```

> **Important:** The `postgres.metrics.enabled: true` flag requires the rayces-v3 Helm
> chart to have the sidecar template added. Confirm with the user that the changes in the
> `rayces-v3` repo (branch `feat/grafana-monitoring-configurations`) have been merged to
> `main` before Flux tries to reconcile this. If not yet merged, temporarily set
> `postgres.metrics.enabled: false` and update after the merge.

---

## Phase 3 — Secrets (Plaintext Templates — User Must Encrypt)

### Step 3.1 — `apps/staging/rayces/secrets/environment-secret.yaml` (PLAINTEXT TEMPLATE)

> **STOP — User must fill in values AND encrypt before committing.**
> Never commit this file in plaintext.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: raycesv3-environment
  namespace: rayces-production
type: Opaque
stringData:
  # ── Core Rails ──────────────────────────────────────────────
  RACK_ENV: "production"
  RAILS_ENV: "production"
  RAILS_LOG_TO_STDOUT: "true"
  RAILS_SERVE_STATIC_FILES: "true"
  RAILS_PORT: "4000"
  SECRET_KEY_BASE: "REPLACE_WITH_SECRET_KEY_BASE"

  # ── Database ─────────────────────────────────────────────────
  POSTGRES_USER: "REPLACE"
  POSTGRES_PASSWORD: "REPLACE"
  POSTGRES_HOST: "postgres-rayces"
  POSTGRES_PORT: "5432"
  POSTGRES_DB_PRODUCTION: "REPLACE"
  POSTGRES_DB_DEVELOPMENT: "REPLACE"
  POSTGRES_DB_TEST: "REPLACE"

  # ── Redis ─────────────────────────────────────────────────────
  REDIS_HOST: "redis-rayces"
  REDIS_PORT: "6379"
  REDIS_PASSWORD: ""

  # ── Application URLs (UPDATE for citas.rayces.com) ───────────
  APP_HOST: "citas.rayces.com"
  APP_PORT: "443"
  FRONTEND_URL: "https://citas.rayces.com"
  NEXTAUTH_URL: "https://citas.rayces.com"
  NEXTAUTH_SECRET: "REPLACE"
  NEXT_PUBLIC_DOMAIN: "citas.rayces.com"
  NEXT_PUBLIC_API_URL: "https://citas.rayces.com/api/v1"
  NEXT_PUBLIC_RAILS_BACKEND: "https://citas.rayces.com"
  NEXT_PUBLIC_RAILS_BACKEND_PROBE: "http://rails-rayces:4000"
  NEXT_PUBLIC_DEFAULT_LOCALE: "es"
  NEXT_PUBLIC_API_LLM_TOKEN: "REPLACE"

  # ── Auth ──────────────────────────────────────────────────────
  GOOGLE_CLIENT_ID: "REPLACE"
  GOOGLE_CLIENT_SECRET: "REPLACE"

  # ── Email / SMTP ──────────────────────────────────────────────
  MAILCATCHER_HOST: ""
  MAILCATCHER_PORT: ""
  MAIL_DOMAIN: "citas.rayces.com"
  DEFAULT_FROM_EMAIL: "REPLACE"
  SMTP_HOST: "REPLACE"
  SMTP_PORT: "587"
  SMTP_USER: "REPLACE"
  SMTP_PASSWORD: "REPLACE"
  SMTP_AUTH: "plain"
  SMTP_STARTTLS: "true"

  # ── Cloudinary ────────────────────────────────────────────────
  CLOUDINARY_CLOUD_NAME: "REPLACE"
  CLOUDINARY_API_KEY: "REPLACE"
  CLOUDINARY_API_SECRET: "REPLACE"

  # ── Security & Rate Limiting ──────────────────────────────────
  RACK_ATTACK_ENABLED: "true"
  PUBLIC_API_RATE_LIMIT: "60"
  BOOKING_RATE_LIMIT: "10"
  TRUSTED_IPS: ""
  SECURITY_TEAM_EMAIL: "REPLACE"

  # ── Feature Flags ─────────────────────────────────────────────
  DEFAULT_LOCALE: "es"
  ALLOW_SEED: "false"
  MULTI_ORG_MIGRATION_MODE: "false"
  WHATSAPP_SIMULATION: "true"
  SKIP_RECAPTCHA: "false"
  RECAPTCHA_SITE_KEY: "REPLACE"
  RECAPTCHA_SECRET_KEY: "REPLACE"

  # ── Impersonation ─────────────────────────────────────────────
  IMPERSONATION_DISABLED: "false"
  IMPERSONATION_DEFAULT_DURATION: "3600"
  IMPERSONATION_MAX_DURATION: "86400"
  IMPERSONATION_STRICT_IP_CHECK: "false"
  NOTIFY_USERS_OF_IMPERSONATION: "true"
  SUPER_ADMIN_REQUIRE_2FA: "false"
  SUPER_ADMIN_SESSION_TIMEOUT: "3600"
```

Encrypt after filling in all values:
```bash
sops --encrypt --in-place apps/staging/rayces/secrets/environment-secret.yaml
```

### Step 3.2 — `apps/staging/rayces/secrets/ghcr-secret.yaml` (PLAINTEXT TEMPLATE)

> **STOP — User must fill in GHCR_PAT and encrypt before committing.**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ghcr-credentials
  namespace: rayces-production
type: kubernetes.io/dockerconfigjson
stringData:
  .dockerconfigjson: |
    {
      "auths": {
        "ghcr.io": {
          "username": "canriquez",
          "password": "REPLACE_WITH_GHCR_PAT",
          "auth": "REPLACE_WITH_BASE64_OF_canriquez:GHCR_PAT"
        }
      }
    }
```

To generate the `auth` base64 value:
```bash
echo -n "canriquez:YOUR_GHCR_PAT" | base64
```

Encrypt after filling in:
```bash
sops --encrypt --in-place apps/staging/rayces/secrets/ghcr-secret.yaml
```

### Step 3.3 — `apps/staging/rayces/secrets/tunnel-secret.yaml` (PLAINTEXT TEMPLATE)

> **STOP — User must fill in tunnel credentials JSON and encrypt before committing.**
> The credentials JSON content comes from `~/.cloudflared/<UUID>.json` (created in
> Prerequisite P1).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tunnel-credentials
  namespace: rayces-production
type: Opaque
stringData:
  credentials.json: |
    REPLACE_WITH_FULL_CONTENTS_OF_TUNNEL_UUID_JSON_FILE
```

To get the contents:
```bash
cat ~/.cloudflared/<TUNNEL-UUID>.json
# Paste the entire JSON object as the value above
```

Encrypt after filling in:
```bash
sops --encrypt --in-place apps/staging/rayces/secrets/tunnel-secret.yaml
```

---

## Phase 4 — Monitoring: ServiceMonitor for postgres_exporter

### Step 4.1 — `monitoring/configs/staging/rayces-postgres-servicemonitor.yaml`

This follows the same pattern as the `postgresql` ServiceMonitor already in the
`monitoring` namespace (used by churlesflows). Confirm the format by reading the
existing churlesflows postgresql ServiceMonitor first.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: postgres-rayces-production
  namespace: monitoring
  labels:
    release: kube-prometheus-stack    # Must match Prometheus serviceMonitorSelector
spec:
  namespaceSelector:
    matchNames:
      - rayces-production
  selector:
    matchLabels:
      app: postgres-rayces            # Must match labels on the metrics Service
      component: metrics
  endpoints:
    - port: http-metrics
      interval: 30s
      scrapeTimeout: 10s
```

> **Dependency:** This ServiceMonitor only works after the `postgres_exporter` sidecar
> and metrics Service are deployed in `rayces-production`. The metrics Service will be
> created by the Helm chart in the `rayces-v3` repo (branch `feat/grafana-monitoring-configurations`).
> Confirm with the user that this chart change is merged to `main` before Flux tries to
> scrape. If not merged yet, Prometheus will simply log a "target not found" warning —
> no harm done.

---

## Phase 5 — Validation Steps

After all files are committed and Flux reconciles (~1-2 minutes), verify in this order:

### Step 5.1 — Flux reconciliation

```bash
# Watch Flux pick up the new kustomization
flux get kustomizations --watch

# Check HelmRelease status
flux get helmreleases -n rayces-production

# If something fails, get the full error
flux logs --kind=HelmRelease --name=rayces --namespace=rayces-production
```

### Step 5.2 — Namespace and pods

```bash
# Namespace should exist
kubectl get namespace rayces-production

# All pods should reach Running state
kubectl get pods -n rayces-production --watch

# postgres-rayces pod should show 2/2 READY (main + metrics sidecar)
kubectl get pod -n rayces-production -l app=postgres
```

### Step 5.3 — Cloudflare tunnel

```bash
# Cloudflared pods should be running
kubectl get pods -n rayces-production -l app=cloudflared

# Verify tunnel is active in Cloudflare dashboard or via:
cloudflared tunnel info rayces-production
```

Access `https://citas.rayces.com` in a browser. Expect the Rayces login page.

### Step 5.4 — Postgres metrics

```bash
# Port-forward to the metrics endpoint
kubectl port-forward svc/postgres-rayces-metrics 9187:9187 -n rayces-production &

# Verify metrics are exposed
curl -s http://localhost:9187/metrics | grep pg_up
# Expected: pg_up 1

curl -s http://localhost:9187/metrics | grep pg_database_size_bytes
# Expected: pg_database_size_bytes{datname="..."} <some number>
```

### Step 5.5 — Prometheus scraping

```bash
# Port-forward to Prometheus
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring &
```

Open http://localhost:9090 and query:
```promql
pg_up{namespace="rayces-production"}
```
Should return `1` within ~60 seconds of Flux deploying the ServiceMonitor.

### Step 5.6 — Confirm rayces-prd is unaffected

```bash
kubectl get pods -n rayces-prd
# Should still show all pods as Running, no changes
```

Access `https://flows.rayces.com` — should still work normally.

---

## Phase 6 — Deploying New Image Versions (Manual Process Until Image Automation)

Since Flux Image Automation controllers are not installed on this cluster, image updates
are done by committing a new tag to this repo:

1. Merge code to `rayces-v3` main
2. GitHub Actions builds and pushes `ghcr.io/canriquez/rayces-backend:<sha>` and
   `ghcr.io/canriquez/rayces-frontend:<sha>`
3. Update the tag values in `apps/staging/rayces/helmrelease.yaml`:
   ```yaml
   images:
     backend:
       tag: "new-sha-here"
     frontend:
       tag: "new-sha-here"
   ```
4. Commit to homelab-cluster main
5. Flux detects the change within 1 minute and rolls out the new pods

This is the interim process. Adding Flux Image Automation (so step 3-4 happen
automatically) is a future enhancement documented separately.

---

## Phase 7 — Cutover and Decommission `rayces-prd` (Future — When Ready)

Only execute this phase after `rayces-production` has been running stably and you are
confident in the setup.

### Option A — Domain swap (recommended)

Update the Cloudflare tunnel routing so `flows.rayces.com` also points to
`rayces-production`:

```bash
# Add flows.rayces.com as an additional hostname in the rayces-production tunnel
cloudflared tunnel route dns rayces-production flows.rayces.com
```

This makes both domains resolve to the new namespace. The old `rayces-prd` cloudflared
deployment continues trying to serve `flows.rayces.com` but Cloudflare will route to
whichever tunnel is active — you can delete the old tunnel after confirming.

### Option B — Data migration (if DB has production data to preserve)

```bash
# Dump from rayces-prd
kubectl exec -n rayces-prd deployment/postgres-rayces -- \
  pg_dump -U $POSTGRES_USER --no-password $POSTGRES_DB \
  > /tmp/rayces_prd_$(date +%Y%m%d).sql

# Restore to rayces-production (stop app pods first to avoid writes)
kubectl scale deployment rails-rayces sidekiq-rayces -n rayces-production --replicas=0
kubectl exec -i -n rayces-production deployment/postgres-rayces -- \
  psql -U $POSTGRES_USER $POSTGRES_DB \
  < /tmp/rayces_prd_$(date +%Y%m%d).sql
kubectl scale deployment rails-rayces sidekiq-rayces -n rayces-production --replicas=2
```

### Decommission

1. Remove `k8s/overlays/production/` directory from `rayces-v3` repo (if it exists)
2. Add deprecation header to `scripts/setup-production.sh` in `rayces-v3` repo
3. Delete the Cloudflare tunnel: `cloudflared tunnel delete rayces-prd`
4. To remove `rayces-prd` from the cluster — since it is NOT Flux-managed, Flux
   will NOT prune it automatically. Delete manually:
   ```bash
   kubectl delete namespace rayces-prd
   ```
5. Update `CLAUDE.md` in homelab-cluster to reflect that both staging and production
   workloads exist on this cluster

---

## File Change Summary

### Files to CREATE in homelab-cluster:

```
apps/staging/rayces/namespace.yaml
apps/staging/rayces/kustomization.yaml          (or entry in clusters/staging/apps.yaml)
apps/staging/rayces/gitrepository.yaml
apps/staging/rayces/helmrelease.yaml
apps/staging/rayces/secrets/environment-secret.yaml    ← encrypt before committing
apps/staging/rayces/secrets/ghcr-secret.yaml           ← encrypt before committing
apps/staging/rayces/secrets/tunnel-secret.yaml         ← encrypt before committing
apps/staging/rayces/secrets/github-secret.yaml         ← encrypt before committing
monitoring/configs/staging/rayces-postgres-servicemonitor.yaml
```

### Files to MODIFY in homelab-cluster:

```
clusters/staging/apps.yaml    ← add rayces entry (if apps are registered explicitly)
CLAUDE.md                      ← update "Single environment" constraint note
```

### Files to CREATE/MODIFY in rayces-v3 (separate repo, separate PR):

```
charts/rayces/values.yaml                              ← add postgres.metrics block
charts/rayces/templates/postgres-deployment.yaml       ← add metrics sidecar
charts/rayces/templates/postgres-metrics-service.yaml  ← new ClusterIP service
charts/rayces/values.rayces-production.yaml            ← new values file
```

> These rayces-v3 changes are tracked in branch `feat/grafana-monitoring-configurations`
> in that repo. They must be merged to `main` before Flux can deploy a working
> `rayces-production` with monitoring enabled.

---

## References

- Churlesflows reference: `apps/staging/churlesflows/` in this repo
- Prometheus ServiceMonitor reference: existing `postgresql` ServiceMonitor in
  `monitoring` namespace (Helm-managed by churlesflows)
- Cloudflare tunnel docs: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/
- Flux HelmRelease docs: https://fluxcd.io/flux/components/helm/helmreleases/
- Flux GitRepository docs: https://fluxcd.io/flux/components/source/gitrepositories/
- SOPS + Age: https://fluxcd.io/flux/guides/mozilla-sops/
