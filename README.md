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
3. Open `https://<dev domain>/` as `admin`. The plugin's dashboards are in the **ClickHouse** folder, ours in **Managed services** (all providers), one folder per provider (**Managed services · Postgres**), **Workspaces** and **Infrastructure**.
4. Point the team's OTLP export (team settings) at `https://<dev domain>/otlp` with the `otlpToken`, so the team collector forwards the resource group's telemetry here.

Plans `201` (Micro, `otel-collector`), `301` (Boost, `grafana`) and `8` (router) in [`ci.yml`](ci.yml); plan IDs differ between installations, so pick plans that exist on the target cluster.

For a 3-instance ClickHouse, set `CLICKHOUSE_TABLE_ENGINE: ReplicatedMergeTree` on `otel-collector` and leave `cluster_name` unset.

## Sending telemetry

```
OTEL_EXPORTER_OTLP_ENDPOINT=https://<dev domain>/otlp
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_EXPORTER_OTLP_HEADERS=Authorization=Bearer <otlpToken>
```

Inside the landscape, use `otel-collector:4317` or `:4318` with the same header.

## What the resource group sends

Everything reaches the landscape through the team collector (`team-collector-<team>` in the o11y namespace) and its OTLP export. Status as of 2026-11-18:

| Source | Signal | Identified by | Status |
|---|---|---|---|
| Workspace agent | pipeline stdout/stderr | log attributes `workspaceId`, `server`, `replicaId`, `stage`, `stepNumber`, `execId`, `source` (`stdout`/`stderr`) | live |
| Workspace router (nginx) | request spans, if tracing is on for the workspace | `ServiceName` `ws-<workspaceId>`, span attributes `server.address`, `url.path`, `http.status_code` | live |
| Marketplace (via its Envoy sidecar) | managed-service events: backups, reconciliation, autoscaling, and a change event whenever health, backups or a resource's utilization changes state | `ServiceName` `marketplace`, `EventName`, log attributes `teamId`, `targetId` (service ID), `name`, `provider`, `creator` | live |
| Marketplace | `codesphere.managed_service.utilization` and `.ratio` gauges | attributes `targetId`, `name`, `provider`, `resource` | codesphere-cloud/codesphere-monorepo#23228 |
| rg Alloy in `rg-<team>` | metrics of PodMonitors/ServiceMonitors (CNPG for Postgres), logs of pods labelled `codesphere.com/collect-logs-to-rg-alloy=true` | attributes `managed-services.codesphere.com/id` and `managed-services.codesphere.com/name` (the service name, sanitized by the provider, codesphere-cloud/codesphere-monorepo#23316) copied from the pod labels; logs also carry `pod`, `container`, `namespace` | behind `rg-o11y-dev`, and only deployed with the OpenSearch platform; for an OTLP export needs codesphere-cloud/codesphere-monorepo#23221 (Alloy in export-only mode) and, for metrics, codesphere-cloud/codesphere-monorepo#23103 |
| Team collector kubeletstats | CPU, memory and volume usage of `rg-<team>` pods | resource attributes `k8s.pod.name`, `k8s.persistentvolumeclaim.name` | codesphere-cloud/codesphere-monorepo#23221 (`rg-o11y-dev`) |

Until those land, the Postgres dashboards only show the marketplace events, and the Postgres alerts have no data (they stay OK); the **Managed services** alerts work already. CNPG metrics and pod logs carry the service ID and its sanitized name, the dashboards show them as `name (ID)` and fall back to the ID for pods without the label; pod and volume usage are joined to services through the `pod` label of the CNPG metrics.

## Dashboards and alerts

Provisioned read-only from [`grafana/provisioning/`](grafana/provisioning/): one folder for what applies to every managed service, one per provider so its dashboards and alert rules stay together. Grafana's file provisioning can't nest folders, hence the `Managed services · ` prefix.

- **Managed services** (every provider, from the marketplace)
  - *Managed services events*: current health, backups and utilization state per service, the event log and the reported utilization, filterable by provider, service, event and severity.
  - Alert rules ([`managed-services.json`](grafana/provisioning/alerting/managed-services.json)): unhealthy, degraded or unknown health, backups overdue, backups started over an hour ago and still not confirmed (with the provider's error, such as wrong backup store credentials), a resource (storage, CPU, …) close to or at its plan value, and error events such as a failed backup or a provider that doesn't answer. Error alerts carry the event, the kind of operation (`operation`: backup, restore, autoscaling, reconciliation) and the cause of the latest error (`cause`) as labels, so the notification says what failed and why.
    The marketplace only logs health, backups and utilization when they change, so these rules look at the latest change of the last 7 days: they resolve with the next change, and a service deleted in a bad state keeps firing until its events expire.
- **Managed services · Postgres** (Postgres, FerretDB and Babelfish)
  - *Postgres overview*: status, activity, storage and replication from CNPG, CPU/memory/disk from kubeletstats, utilization the provider reports.
  - *Postgres logs and events*: marketplace events and pod logs, with Postgres messages parsed out of CNPG's JSON lines.
  - Alert rules ([`managed-services-postgres.json`](grafana/provisioning/alerting/managed-services-postgres.json)), from CNPG and kubeletstats: down, connections over 80%, replication lag over 1 min, WAL archiving failing, transaction ID age over 1 billion, a volume over 90% full, a transaction open for over an hour.
- **Workspaces**
  - *Workspaces overview*: requests, 5xx share, latency and log volume per workspace.
  - *Workspace requests*: router spans by server, status class and path, with the slowest and failed requests.
  - *Workspace logs*: pipeline output by server and stage.
  - Alert rules ([`workspaces.json`](grafana/provisioning/alerting/workspaces.json)): from the router spans, over 10% 5xx responses per workspace server for 5 minutes, and a server the router couldn't reach (502, 503, 504) in 3 or more minutes of the last 30, which is how a process that keeps crashing, for example OOM-killed, shows; over 5% failed server spans per service that sends its own traces.
    Workspaces only send router spans with tracing on, and nothing reports an OOM kill as such yet.
- **Infrastructure**
  - *Service Infrastructure Metrics*: CPU, memory, network, disk I/O, requests, limits and volumes per pod of the resource group. 
A new provider gets a `dashboards/managed-services/<provider>/` directory, a provider entry in [`folders.yaml`](grafana/provisioning/dashboards/folders.yaml) with folder `Managed services · <Provider>`, and an `alerting/managed-services-<provider>.json` with the same folder title.

No contact point is provisioned; add one in Grafana to get notified. Grafana keeps the **Codesphere** folder of earlier versions after an upgrade, empty; delete it by hand.

## Not verified on a cluster

- Managed-service hostnames (`ms-<provider>-<schemaVersion>-<team>-landscape-<workspace>-<name>`).
- `ReplicatedMergeTree` against 3 instances.
- Grafana as UID 472, plugin install from grafana.com at startup, Postgres TLS.
- The dashboards and alerts were checked against a local ClickHouse, the collector above and Grafana 13.2.6 with synthetic telemetry shaped like each source; the names of the kubeletstats and utilization metrics follow the open PRs in the table.
