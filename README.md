# OpenTelemetry Collector + Grafana Stack

A containerized observability stack using **OpenTelemetry Collector**, **Grafana**, and supporting backends for metrics, traces, and logs.

## Architecture

```
Your Application
   │
   │  OTLP (gRPC :4317 / HTTP :4318)
   ▼
┌──────────────────────┐
│  OTel Collector      │
│  (receive/process/   │
│   export telemetry)  │
└──┬───────┬───────┬───┘
   │       │       │
   ▼       ▼       ▼
Prometheus  Tempo   Loki
(metrics)  (traces) (logs)
   │       │       │
   └───────┼───────┘
           ▼
       Grafana
    (visualization)
```

## Components

| Service         | Image                                    | Port  | Purpose            |
|-----------------|------------------------------------------|-------|---------------------|
| OTel Collector  | `otel/opentelemetry-collector-contrib`   | 4317, 4318 | Telemetry ingestion |
| Prometheus      | `prom/prometheus`                        | 9090  | Metrics backend     |
| Tempo           | `grafana/tempo`                          | 3200  | Traces backend      |
| Loki            | `grafana/loki`                           | 3100  | Logs backend        |
| Grafana         | `grafana/grafana`                        | 3000  | Dashboards & UI     |

## Quick Start

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/install/) installed

### Start the stack

```bash
docker compose up -d
```

### Verify everything is running

```bash
docker compose ps
```

### Access the UIs

| UI              | URL                          | Credentials     |
|-----------------|------------------------------|-----------------|
| **Grafana**     | http://localhost:3000         | admin / admin   |
| **Prometheus**  | http://localhost:9090         |                 |
| **OTel zPages** | http://localhost:55679/debug  |                 |

### Stop the stack

```bash
docker compose down
```

### Stop and remove all data

```bash
docker compose down -v
```

## Sending Telemetry

Configure your application to send OpenTelemetry data to the collector:

| Protocol  | Endpoint                    |
|-----------|-----------------------------|
| OTLP gRPC | `http://localhost:4317`     |
| OTLP HTTP | `http://localhost:4318`     |

### Environment variables for your app

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4317"
export OTEL_EXPORTER_OTLP_PROTOCOL="grpc"
export OTEL_SERVICE_NAME="my-service"
```

### Example: Python app with auto-instrumentation

```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install
opentelemetry-instrument --service_name my-service python app.py
```

### Example: Node.js app

```bash
npm install @opentelemetry/auto-instrumentations-node @opentelemetry/exporter-trace-otlp-grpc
```

```javascript
// tracing.js
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({ url: 'http://localhost:4317' }),
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();
```

## Grafana Datasources (Pre-configured)

All datasources are automatically provisioned with cross-linking:

- **Prometheus** — Query metrics, linked to Tempo for exemplar traces
- **Tempo** — Query traces, linked to Prometheus for span metrics and Loki for correlated logs
- **Loki** — Query logs, linked to Tempo for trace correlation

## Project Structure

```
├── docker-compose.yml
├── otel-collector/
│   └── otel-collector-config.yaml
├── prometheus/
│   └── prometheus.yml
├── tempo/
│   └── tempo-config.yaml
├── loki/
│   └── loki-config.yaml
└── grafana/
    ├── dashboards/
    │   └── otel-collector-dashboard.json
    └── provisioning/
        ├── dashboards/
        │   └── dashboards.yaml
        └── datasources/
            └── datasources.yaml
```

## Health Checks

- **OTel Collector:** http://localhost:13133
- **Prometheus:** http://localhost:9090/-/healthy
- **Loki:** http://localhost:3100/ready
- **Tempo:** http://localhost:3200/ready
- **Grafana:** http://localhost:3000/api/health
