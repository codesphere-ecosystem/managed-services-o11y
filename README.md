# ClickHouse observability landscape

A Codesphere landscape that ingests OpenTelemetry logs, traces and metrics, stores them in a managed ClickHouse, and shows them in Grafana.

```
app ──OTLP/HTTP──▶ gateway /otlp ──▶ otel-collector ──native 9000──▶ clickhouse (managed)
                   gateway /     ──▶ grafana ─────────native 9000──▶ clickhouse
                                     grafana ──────────5432─────────▶ grafana-db (managed Postgres)
```

| Service | Kind | Purpose |
|---|---|---|
| `gateway` | router | `/otlp` → collector (OTLP/HTTP 4318), `/` → Grafana. |
| `otel-collector` | container | OTLP gRPC 4317 / HTTP 4318 with bearer-token auth, `clickhouse` exporter for all signals. Config: [`otel/config.yaml`](otel/config.yaml). |
| `grafana` | container | ClickHouse datasource and the plugin's bundled dashboards. Provisioning: [`grafana/provisioning/`](grafana/provisioning/). |
| `clickhouse` | managed (`clickhouse` v0) | Telemetry store. Tables `otel_logs`, `otel_traces`, `otel_metrics_*` in database `app`, TTL `TELEMETRY_TTL` (7 days). |
| `grafana-db` | managed (`postgres` v1) | Grafana's state; the container disk doesn't survive restarts. |

## Deploy

1. Create a workspace from this repo.
2. Sync the landscape and fill the vault secrets: `clickhousePassword` (8–32 alphanumeric), `otlpToken`, `grafanaAdminPassword`, `grafanaDbPassword`, `grafanaDbSuperuserPassword`.
3. Open `https://<dev domain>/` as `admin`. The plugin's dashboards are in the **ClickHouse** folder, ours and the alert rules in **Codesphere**.

Plans `21` (containers) and `8` (router) in [`ci.yml`](ci.yml) are placeholders; pick plans that exist on the target cluster.

For a 3-instance ClickHouse, set `CLICKHOUSE_TABLE_ENGINE: ReplicatedMergeTree` on `otel-collector` and leave `cluster_name` unset.

## Sending telemetry

```
OTEL_EXPORTER_OTLP_ENDPOINT=https://<dev domain>/otlp
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_EXPORTER_OTLP_HEADERS=Authorization=Bearer <otlpToken>
```

Inside the landscape, use `otel-collector:4317` or `:4318` with the same header.

## Dashboards and alerts

[`grafana/provisioning/dashboards/codesphere/`](grafana/provisioning/dashboards/codesphere/) and [`grafana/provisioning/alerting/rules.json`](grafana/provisioning/alerting/rules.json), read-only in Grafana:

- **Managed services**: CloudNativePG metrics and managed-service pod logs that the resource group's Alloy sends, keyed by `managed-services.codesphere.com/id`.
- **Workspaces**: pipeline stdout/stderr that workspace agents send (`workspaceId`, `server`, `replicaId`, `stage` log attributes).
- **Alerts**: Postgres down, connections over 80%, replication lag over 1 min, WAL archiving failing, transaction ID age over 1 billion, and over 5% failed server spans per traced service. No backup alert: CNPG's backup metrics don't track plugin backups. No contact point is provisioned; add one in Grafana to get notified.

## Not verified on a cluster

- Managed-service hostnames (`ms-<provider>-<schemaVersion>-<team>-landscape-<workspace>-<name>`).
- `ReplicatedMergeTree` against 3 instances.
- Grafana as UID 472, plugin install from grafana.com at startup, Postgres TLS.
