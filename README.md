# Codesphere Observability Stack

A Codesphere landscape that brings the logs, metrics and traces of a resource group's workspaces and managed services into one place. An OpenTelemetry Collector receives OTLP and writes it to a managed ClickHouse, and Grafana, with its state in a managed Postgres, comes with dashboards and alert rules for managed services, workspaces and infrastructure. To use it, point the resource group's OTLP export at the *Collector endpoint (OTLP/HTTP)* from the service details, with the `otlpToken` as auth token.

## License

This repository is licensed under [Apache-2.0](LICENSE). It holds configuration only and runs these components unmodified (image config adjusted, upstream layers unmodified):

| Component | License | Source |
|---|---|---|
| OpenTelemetry Collector Contrib | Apache-2.0 | [open-telemetry/opentelemetry-collector-contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib) |
| Grafana | [AGPL-3.0](LICENSES/AGPL-3.0.txt) | [grafana/grafana](https://github.com/grafana/grafana) |
| Grafana ClickHouse data source | Apache-2.0 | [grafana/clickhouse-datasource](https://github.com/grafana/clickhouse-datasource) |

Versions and details are in [NOTICE](NOTICE).

ClickHouse® is a registered trademark of ClickHouse, Inc. Grafana is a trademark of Raintank, Inc. dba Grafana Labs. OpenTelemetry is a trademark of The Linux Foundation.
