# Grafana SMTP — Email Notifications via Brevo

## Overview

Grafana is configured to send alert notification emails through **Brevo** (formerly Sendinblue), a transactional email service. Emails originate from the `experientialabs.net` domain (managed in Cloudflare), authenticated with DKIM and DMARC, and are delivered via Brevo's shared SMTP relay.

---

## Architecture

```
Grafana Pod (monitoring namespace)
  │
  │  grafana.ini [smtp] section
  │  password resolved from env var → $__env{SMTP_PASSWORD}
  │
  │  env var injected from Kubernetes Secret
  │  "grafana-smtp-credentials" (SOPS-encrypted in Git)
  │
  ▼
Brevo SMTP Relay
  smtp-relay.brevo.com:587 (STARTTLS)
  SMTP login: injected via $__env{SMTP_USER} (stored in grafana-smtp-credentials)
  │
  ▼
Delivered as:
  From: ElBot Grafana Monitoring <bot@experientialabs.net>
  To: recipient configured per alert rule
```

---

## Involved Services

### 1. Grafana (kube-prometheus-stack)
- Deployed via the `kube-prometheus-stack` HelmRelease in the `monitoring` namespace
- SMTP is configured through the `grafana.ini` section inside the Helm values in `release.yaml`
- The SMTP password is **never stored in plaintext** — it is read at runtime from an environment variable injected by a Kubernetes Secret

### 2. Brevo (transactional email provider)
- Account: SMTP username stored in the `grafana-smtp-credentials` secret (key `SMTP_USER`) — never in plaintext
- SMTP relay: `smtp-relay.brevo.com:587` with MandatoryStartTLS
- The sending domain `experientialabs.net` is fully authenticated on the Brevo account:
  - **DKIM signature**: configured and verified (`experientialabs.net`)
  - **DMARC**: configured and verified
  - **Shared IP**: Brevo shared sending infrastructure
- Verified sender: `ElBot <bot@experientialabs.net>`

### 3. Cloudflare (DNS for experientialabs.net)
- The `experientialabs.net` domain is parked and managed in Cloudflare
- Brevo domain authentication required adding the following DNS record types to Cloudflare:
  - **SPF** (`TXT` on `@`): authorises Brevo's servers to send on behalf of the domain
  - **DKIM** (`TXT` on `mail._domainkey` or equivalent): cryptographic signature for outgoing mail
  - **DMARC** (`TXT` on `_dmarc`): policy record instructing receivers how to handle unauthenticated mail

### 4. Kubernetes Secret — `grafana-smtp-credentials`
- Namespace: `monitoring`
- Holds (all values SOPS-encrypted, key names in plaintext):
  - `SMTP_PASSWORD` — the Brevo SMTP API token
  - `SMTP_USER` — the Brevo SMTP login username
- Injected via `envFromSecret` and read by `grafana.ini` as `$__env{SMTP_USER}` / `$__env{SMTP_PASSWORD}`
- Encrypted at rest in Git using **SOPS + Age**
  - File: `monitoring/configs/staging/kube-prometheus-stack/grafana-smtp-secret.yaml`
  - Age recipient: `age1c8uz8qzz095uspf99rfpketpcl9fk3zpwtjkts0nx4tyxctjqdyspqzj36`
- Decrypted and applied to the cluster by the `monitoring-configs` Flux Kustomization (SOPS provider configured)

### 5. Kubernetes Secret — `grafana-admin-credentials`
- Namespace: `monitoring`
- Holds (all values SOPS-encrypted, key names in plaintext):
  - `admin-user` — Grafana admin username (`churles`)
  - `admin-password` — Grafana admin password (must be a value you know — set it, do not leave random)
- Wired via the chart's **`admin.existingSecret`** (`userKey: admin-user`, `passwordKey: admin-password`), NOT `envFromSecret`.
  - Why: the Grafana chart always sets `GF_SECURITY_ADMIN_USER`/`GF_SECURITY_ADMIN_PASSWORD` as explicit `env:` (`valueFrom`), and Kubernetes `env:` overrides `envFrom:`. So admin creds supplied through `envFromSecret` are silently ignored — `admin.existingSecret` is the only mechanism that works.
- File: `monitoring/configs/staging/kube-prometheus-stack/grafana-admin-secret.yaml`

---

## Configuration

### Helm Values (`monitoring/controllers/base/kube-prometheus-stack/release.yaml`)

```yaml
grafana:
  envFromSecret: grafana-smtp-credentials   # injects all secret keys as env vars into the Grafana pod

  grafana.ini:
    smtp:
      enabled: true
      host: smtp-relay.brevo.com:587
      user: $__env{SMTP_USER}               # Brevo SMTP login, injected from the secret
      from_address: bot@experientialabs.net  # verified sender on Brevo
      from_name: ElBot Grafana Monitoring
      startTLS_policy: MandatoryStartTLS
      password: $__env{SMTP_PASSWORD}        # resolved from the injected secret at runtime
```

### Secret (`monitoring/configs/staging/kube-prometheus-stack/grafana-smtp-secret.yaml`)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: grafana-smtp-credentials
  namespace: monitoring
data:
  SMTP_PASSWORD: <SOPS-encrypted Brevo SMTP API token>
  SMTP_USER: <SOPS-encrypted Brevo SMTP login>
  GF_SECURITY_ADMIN_USER: <SOPS-encrypted Grafana admin user>
  GF_SECURITY_ADMIN_PASSWORD: <SOPS-encrypted Grafana admin password>
```

The secret is committed encrypted. To rotate the token:
1. Get the new SMTP API token from Brevo → SMTP & API → API Keys
2. Decrypt the file: `sops --decrypt --in-place grafana-smtp-secret.yaml`
3. Replace the base64-encoded value of `SMTP_PASSWORD`
4. Re-encrypt: `sops --encrypt --in-place grafana-smtp-secret.yaml`
5. Commit the encrypted file — Flux will reconcile automatically

---

## How Secret Injection Works

The `envFromSecret` field in the Grafana Helm values tells the kube-prometheus-stack chart to pass all keys from the named Kubernetes Secret as environment variables into the Grafana container. Grafana's `grafana.ini` supports `$__env{VAR_NAME}` interpolation, which reads the value of the named env var at startup — keeping the plaintext password out of both Git and the Helm values.

```
grafana-smtp-credentials (Secret)
  └── SMTP_PASSWORD=<token>
        │
        │  envFromSecret
        ▼
  Grafana container env
        │
        │  $__env{SMTP_PASSWORD}
        ▼
  grafana.ini → smtp.password
```

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| SMTP test fails with "authentication failed" | Secret not injected — check `envFromSecret` spelling in release.yaml, verify secret exists in `monitoring` namespace |
| Brevo rejects with "sender not valid" | `from_address` is not a verified sender in Brevo — must match a verified email or authenticated domain |
| Email sent but not delivered | SPF/DKIM/DMARC DNS records missing or not propagated in Cloudflare |
| Secret value not updated after rotation | Flux may need a reconcile trigger — run `flux reconcile kustomization monitoring-configs` |

---

## Key File Locations

| Purpose | Path |
|---|---|
| Grafana Helm values (SMTP config) | `monitoring/controllers/base/kube-prometheus-stack/release.yaml` |
| SMTP credentials secret (encrypted) | `monitoring/configs/staging/kube-prometheus-stack/grafana-smtp-secret.yaml` |
| Flux Kustomization (applies secrets) | `clusters/staging/monitoring.yaml` → `monitoring-configs` |
