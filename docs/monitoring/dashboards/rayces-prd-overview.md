# Grafana Dashboard — Rayces-PRD Production Overview

## Overview

This document captures the configuration of the **Rayces-PRD Production Overview** dashboard in Grafana, including all panels, alert rules, and notification routing. This dashboard is the primary observability surface for the `rayces-prd` namespace workloads.

- **Dashboard UID:** `rayces-prd-overview`
- **URL path:** `/d/rayces-prd-overview`
- **Persistence:** Stored in Grafana's database — survives pod restarts
- **Auto-refresh:** Every 30 seconds

---

## Monitored Workloads

The dashboard covers all containers running in the `rayces-prd` namespace:

| Workload | Type |
|---|---|
| `rails-rayces` | Rails web application |
| `nextjs-rayces` | Next.js frontend |
| `sidekiq-rayces` | Background job processor |
| `redis-rayces` | In-memory cache |
| `postgres-rayces` | PostgreSQL database |
| `cloudflared` | Cloudflare tunnel daemon |

---

## Dashboard Sections & Panels

### 1. Service Health
High-level readiness and stability signals.

| Panel | Description |
|---|---|
| Pod Ready Status | Live ready/not-ready state per pod |
| Container Restarts (1h) | Restart count per container over the last hour |
| Running Pod Count | Total running pods in the namespace |

### 2. CPU Performance
CPU consumption and throttling by container.

| Panel | Description |
|---|---|
| CPU Usage by Container | CPU cores consumed per container (rate) |
| CPU Throttling % | Percentage of CPU time throttled per container |

### 3. Memory
Memory footprint per container using two complementary metrics.

| Panel | Description |
|---|---|
| Working Set Memory | Active memory pages in use (what the kernel counts against limits) |
| RSS Memory | Resident set size — physical memory pages not swappable |

### 4. Disk & Storage
PersistentVolumeClaim (PVC) usage across the namespace.

| Panel | Description |
|---|---|
| PVC Disk Used % (gauge) | Current used percentage per PVC |
| PVC Available | Raw available space in GiB (baseline: 37.8 GiB free at setup) |
| PVC Used Over Time | Historical trend of PVC utilisation |

### 5. Network Traffic
Inbound and outbound bytes per second per pod.

| Panel | Description |
|---|---|
| Inbound bytes/s | Network receive rate per pod |
| Outbound bytes/s | Network transmit rate per pod |

### 6. PostgreSQL
Container-level metrics for the `postgres-rayces` container.

| Panel | Description |
|---|---|
| PostgreSQL CPU | CPU usage for the postgres container |
| PostgreSQL Memory | Memory usage for the postgres container |
| PostgreSQL Disk I/O | Disk read/write throughput for the postgres container |

> **Note:** These are container-level cAdvisor metrics, not database-level metrics. `postgres-rayces` does not currently have a `pg_exporter` sidecar. If a `pg_exporter` sidecar is added in future, these panels should be replaced with DB-level metrics (active connections, query rate, lock waits, etc.).

### 7. Redis
Memory and CPU usage for the `redis-rayces` container.

| Panel | Description |
|---|---|
| Redis Memory | Memory consumed by the Redis container |
| Redis CPU | CPU usage for the Redis container |

### 8. Workload Availability
Deployment health and OOM events.

| Panel | Description |
|---|---|
| Available Replicas per Deployment | Current available replica count per Deployment |
| OOM Events | Container OOM kill events detected in the namespace |

---

## Alert Rules

All rules live in the **"Rayces-PRD Alerts"** Grafana folder. Alerts are routed to the **email receiver** contact point for all labels matching `namespace=rayces-prd`.

| Rule | Condition | Severity | Pending period |
|---|---|---|---|
| Pod Not Ready | Any pod drops out of Ready state | Critical | 2 minutes |
| High Container Restart Rate | Any container restarts > 3 times/hour | Warning | immediate |
| High CPU Throttling | CPU throttling > 80% | Warning | 5 minutes |
| PVC Disk Usage High | Any PVC usage > 80% | Warning | 5 minutes |
| OOM Kill Detected | Any container is OOM-killed | Critical | immediate |
| Deployment Has Zero Available Replicas | Any Deployment drops to 0 available replicas | Critical | 2 minutes |

---

## Notification Routing

- **Contact point:** Email receiver (Brevo SMTP via `bot@experientialabs.net`)
- **Routing policy:** All alerts with label `namespace=rayces-prd` are routed directly to the email contact point
- For SMTP configuration details see: [`docs/monitoring/grafana-smtp.md`](../grafana-smtp.md)

---

## Prometheus Metric Sources

The dashboard queries metrics collected by the `kube-prometheus-stack` Prometheus instance scraping the `rayces-prd` namespace. Key metric families used:

| Metric family | Used for |
|---|---|
| `kube_pod_status_ready` | Pod ready status |
| `kube_pod_container_status_restarts_total` | Container restart counts |
| `container_cpu_usage_seconds_total` | CPU usage |
| `container_cpu_cfs_throttled_seconds_total` | CPU throttling |
| `container_memory_working_set_bytes` | Working set memory |
| `container_memory_rss` | RSS memory |
| `kubelet_volume_stats_used_bytes` / `kubelet_volume_stats_capacity_bytes` | PVC usage |
| `container_network_receive_bytes_total` / `container_network_transmit_bytes_total` | Network traffic |
| `kube_deployment_status_replicas_available` | Available replicas |
| `kube_pod_container_status_last_terminated_reason` | OOM kills |

---

## Maintenance Notes

- The dashboard is stored in Grafana's internal SQLite/PostgreSQL DB and is not version-controlled as a ConfigMap or JSON file. If the Grafana pod is re-provisioned from scratch (PVC deleted), the dashboard will need to be recreated.
- To make the dashboard GitOps-managed in future, export it as JSON and provision it via a Grafana dashboard ConfigMap in the `monitoring` namespace.
