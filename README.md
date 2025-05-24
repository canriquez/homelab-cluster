# PostgreSQL Service Setup for `churlesflows`

This guide documents the full process of installing, managing, and cleaning up a Bitnami PostgreSQL Helm chart deployment using FluxCD and SOPS, inside a structured GitOps repository.

---

## 📦 PostgreSQL Overview

[PostgreSQL](http://www.postgresql.org/) is a powerful, open-source object-relational database system. Bitnami provides a Helm chart to install PostgreSQL in Kubernetes.

* Chart repo: [https://artifacthub.io/packages/helm/bitnami/postgresql](https://artifacthub.io/packages/helm/bitnami/postgresql)
* Source: [https://github.com/bitnami/charts/tree/main/bitnami/postgresql](https://github.com/bitnami/charts/tree/main/bitnami/postgresql)

---

## 🗂 Folder Structure

Organize files for the `churlesflows` PostgreSQL service like this:

```
apps/
├── base
│   └── churlesflows
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       ├── postgres
│       │   ├── helmrelease.yaml
│       │   └── repository.yaml
│       └── secrets
│           └── postgres-churlesflow-secret.yaml
└── staging
    └── churlesflows
        └── kustomization.yaml
```

---

## 🧾 Show Available Chart Values

```bash
helm show values oci://registry-1.docker.io/bitnamicharts/postgresql --version 16.7.4
```

---

## 🧭 Create Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: churlesflows
```

---

## 🔐 Generate and Encrypt Credentials Secret

Use a secret that contains credentials under `values.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-churlesflow-secret
  namespace: churlesflows
type: Opaque
stringData:
  values.yaml: |
    auth:
      username: churles
      password: <password>
      postgresPassword: <password>
      database: n8n-churlesflow
```

### One-liner:

```bash
kubectl create secret generic postgres-churlesflow-secret \
  --namespace=churlesflows \
  --from-literal=values.yaml=$'auth:\n  username: churles\n  password: <password>\n  postgresPassword: <password>\n  database: n8n-churlesflow' \
  --dry-run=client -o yaml > postgres-churlesflow-secret.yaml
```

### Encrypt with SOPS:

```bash
sops --age=$AGE_PUBLIC \
--encrypt --encrypted-regex '^(data|stringData)$' --in-place postgres-churlesflow-secret.yaml
```

### Move to GitOps repo:

```bash
mv postgres-churlesflow-secret.yaml apps/base/churlesflows/secrets/
```

---

## 🔗 Define the HelmRepository

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: bitnami-postgresql
  namespace: churlesflows
spec:
  type: oci
  interval: 24h
  url: oci://registry-1.docker.io/bitnamicharts
```

---

## 🚀 Deploy PostgreSQL with HelmRelease

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: postgresql
  namespace: churlesflows
spec:
  interval: 30m
  chart:
    spec:
      chart: postgresql
      version: 16.7.x
      sourceRef:
        kind: HelmRepository
        name: bitnami-postgresql
        namespace: churlesflows
      interval: 12h
  install:
    remediation:
      retries: 3
  valuesFrom:
    - kind: Secret
      name: postgres-churlesflow-secret
      valuesKey: values.yaml
  values:
    primary:
      persistence:
        enabled: true
        size: 20Gi
```

---

## 🧩 Kustomization for PostgreSQL

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - ./postgres/repository.yaml
  - ./postgres/helmrelease.yaml
  - ./secrets/postgres-churlesflow-secret.yaml
```

---

## 🔐 Users and Roles

### `postgres` user

* Superuser
* Password in `postgresql` secret

### `churles` user

* Created on first boot
* Password from your secret
* Owns `n8n-churlesflow` database

---

## ✅ How to Log In

### As `postgres`

```bash
psql -h 127.0.0.1 -p 5433 -U postgres
```

### As `churles`

```bash
psql -h 127.0.0.1 -p 5433 -U churles -d n8n-churlesflow
```

### Test Permissions

```sql
-- Connect to DB and test
\c n8n-churlesflow
CREATE TABLE test (id SERIAL PRIMARY KEY, msg TEXT);
INSERT INTO test (msg) VALUES ('Hello from churles!');
SELECT * FROM test;
```

---

## 🔄 Password Rotation with passwordUpdateJob

Create a new secret with `passwordUpdateJob.enabled: true`

```bash
kubectl create secret generic postgres-churlesflow-secret \
  --namespace=churlesflows \
  --from-literal=values.yaml=$'auth:\n  username: churles\n  password: <newpassword>\n  postgresPassword: <newpassword>\n  database: n8n-churlesflow\npasswordUpdateJob:\n  enabled: true' \
  --dry-run=client -o yaml > postgres-churlesflow-secret.yaml

sops --age=$AGE_PUBLIC -e -i postgres-churlesflow-secret.yaml
mv postgres-churlesflow-secret.yaml apps/base/churlesflows/secrets/
```

Commit and reconcile:

```bash
git add .
git commit -m "Rotate churles password"
git push
flux reconcile kustomization apps -n flux-system
```

---

## 🧹 Cleanup Commands

### Delete HelmRelease

```bash
kubectl delete helmrelease postgresql -n churlesflows --ignore-not-found
```

### Delete PVCs

```bash
kubectl delete pvc -l app.kubernetes.io/instance=postgresql -n churlesflows --ignore-not-found
```

### Delete Pods

```bash
kubectl delete pod -l app.kubernetes.io/instance=postgresql -n churlesflows --force --grace-period=0
```

### Delete Services

```bash
kubectl delete svc postgresql postgresql-hl -n churlesflows --ignore-not-found
```

### Delete Namespace (optional)

```bash
kubectl delete ns churlesflows --ignore-not-found
```

### Delete Secret from Git (optional)

```bash
rm apps/base/churlesflows/postgres/secrets/postgres-churlesflow-secret.yaml
```

---

This document serves as a full reference for managing the `churlesflows` PostgreSQL service in a GitOps-managed Kubernetes cluster using FluxCD, Helm, and SOPS.

