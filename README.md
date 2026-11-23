# Codesphere Observability Stack

A Codesphere landscape that brings the logs, metrics and traces of a resource group's workspaces and managed services into one place. An OpenTelemetry Collector receives OTLP and writes it to a managed ClickHouse, and Grafana, with its state in a managed Postgres, comes with dashboards and alert rules for managed services, workspaces and infrastructure. To use it, point the resource group's OTLP export at the collector endpoint from the service details, with the `otlpToken` as auth token.
