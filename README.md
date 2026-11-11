# ClickHouse observability landscape

A Codesphere landscape that ingests OpenTelemetry logs, traces and metrics, stores them in a managed ClickHouse, and shows them in Grafana.

```
app ──OTLP/HTTP──▶ gateway /otlp ──▶ otel-collector ──native 9000──▶ clickhouse (managed)
                   gateway /     ──▶ grafana ─────────native 9000──▶ clickhouse
                                     grafana ──────────5432─────────▶ grafana-db (managed Postgres)
```

| Service | Kind | Purpose |
|---|---|---|
| `gateway` | router | Public entry: `/otlp` → collector (OTLP/HTTP 4318), `/` → Grafana. |
| `otel-collector` | container runtime | OTLP receivers (gRPC 4317 in the landscape, HTTP 4318 public), bearer-token auth, `clickhouse` exporter for all three signals. Config: `otel/config.yaml`. |
| `grafana` | container runtime | ClickHouse datasource with the OTel logs/traces settings, plus the plugin's bundled dashboards (OTel service, traces and logs explorers, ClickHouse monitoring). Provisioning: `grafana/provisioning/`. |
| `clickhouse` | managed (`clickhouse` v0, Small) | Telemetry store. The exporter creates `otel_logs`, `otel_traces`, `otel_metrics_*` in database `app`; rows expire after `TELEMETRY_TTL` (7 days). |
| `grafana-db` | managed (`postgres` v1) | Grafana's own database (users, saved dashboards, alerts). The container's disk is lost on restart, so SQLite there would forget everything. |

## Deploy

1. Push this folder to a repo and create a workspace from it.
2. Sync the landscape. Codesphere asks for the vault secrets: `clickhousePassword` (8–32 alphanumeric), `otlpToken`, `grafanaAdminPassword`, `grafanaDbPassword`, `grafanaDbSuperuserPassword`.
3. Open `https://<dev domain>/` and log in as `admin`. Dashboards are in the **ClickHouse** folder; Explore → ClickHouse shows logs and traces.

Plans `21` (containers) and `8` (router) are placeholders; pick plans that exist on the target cluster.

## Sending telemetry

```
OTEL_EXPORTER_OTLP_ENDPOINT=https://<dev domain>/otlp
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_EXPORTER_OTLP_HEADERS=Authorization=Bearer <otlpToken>
```

Services in the same landscape can also use `otel-collector:4317` (gRPC) or `:4318`, with the same header.

## 3-instance ClickHouse

For the Medium or Large plan, set `CLICKHOUSE_TABLE_ENGINE: ReplicatedMergeTree` on `otel-collector`. With 3 instances `app` is a `Replicated` database that refuses unreplicated tables. Leave `cluster_name` unset: `ON CLUSTER` is not allowed in a `Replicated` database.

## Local smoke test

`local/compose.yaml` runs the same collector config and Grafana provisioning against a plain ClickHouse:

```
cd local && docker compose up -d
docker run --rm --network obs-landscape_default ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:v0.165.0 \
  traces --otlp-endpoint otel-collector:4318 --otlp-http --otlp-insecure \
  --otlp-header 'Authorization="Bearer local-token"' --duration 3s
```

Grafana: http://localhost:3000 (`admin` / `local-admin`).

## Verified

- `ci.yml` parses with the monorepo's pipeline parser (`parsePipelineConfig`, schema v0.5).
- Locally: traces, logs and metrics from telemetrygen land in `otel_traces`, `otel_logs`, `otel_metrics_gauge`. OTLP without the token gets 401. The Grafana datasource health check passes, the 8 bundled dashboards are provisioned, and SQL through Grafana returns all three signals.

## Not verified

- A real deploy: ClickHouse isn't on a Codesphere cluster yet.
- The managed-service hostnames are derived from the marketplace's naming (`ms-<provider>-<schemaVersion>-<team>-landscape-<workspace>-<name>`) and ClickHouse living in `rg-<team>`.
- The `ReplicatedMergeTree` setting against a 3-instance service.
- Grafana as UID 472 on Codesphere, and Grafana reaching grafana.com to install the plugin at startup.
- Postgres TLS (`GF_DATABASE_SSL_MODE: require`).
