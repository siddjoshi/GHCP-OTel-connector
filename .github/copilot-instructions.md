# Copilot Instructions

## Architecture

This is a Docker Compose-based **OpenTelemetry observability stack** — no application code, only infrastructure config. The data flow is:

```
Application → OTel Collector (receive/process/export) → Prometheus (metrics)
                                                      → Tempo (traces)
                                                      → Loki (logs)
                                                      → Grafana (visualization)
```

The OTel Collector is the single ingestion point. Applications send OTLP data to it (gRPC `:4317` / HTTP `:4318`), and it fans out to the three backends. Grafana datasources are cross-linked: Prometheus ↔ Tempo ↔ Loki, enabling seamless navigation between metrics, traces, and logs.

## Running the Stack

```bash
docker compose up -d      # start
docker compose ps          # verify
docker compose down        # stop
docker compose down -v     # stop and remove all data
```

Grafana is exposed on **port 3010** (mapped from container port 3000). Default credentials: `admin` / `admin`.

## Key Configuration Files

- **`otel-collector/otel-collector-config.yaml`** — Defines receivers, processors, exporters, and pipelines. All three signal types (traces, metrics, logs) use the same `otlp` receiver with `memory_limiter` → `batch` processing chain.
- **`grafana/provisioning/datasources/datasources.yaml`** — Auto-provisions all three datasources with cross-linking (exemplar trace IDs, trace-to-logs, trace-to-metrics, service maps).
- **`grafana/dashboards/*.json`** — Pre-built dashboards auto-provisioned into an "OpenTelemetry" folder.

## Conventions

- **All config files are mounted read-only** (`:ro`) into containers. Edit the local files, then recreate the container (`docker compose up -d <service>`).
- Datasource UIDs are hardcoded (`prometheus`, `tempo`, `loki`) and referenced across datasource configs and dashboard JSON. Keep these stable when editing.
- Prometheus receives metrics via **remote write** (not scrape from OTel Collector). The `--web.enable-remote-write-receiver` flag enables this. Prometheus also scrapes its own metrics and the collector's internal metrics on port `8888`.
- Tempo generates span-metrics and service-graphs, writing them back to Prometheus via remote write with exemplars enabled.
- Loki has `allow_structured_metadata: true` to support OpenTelemetry log attributes.
- Grafana feature toggles `traceqlEditor`, `traceQLStreaming`, and `metricsSummary` are enabled for advanced Tempo querying.

## Adding a New Dashboard

Add a JSON file to `grafana/dashboards/`. It will be auto-provisioned on Grafana restart. Use the stable datasource UIDs (`prometheus`, `tempo`, `loki`) in dashboard JSON.
