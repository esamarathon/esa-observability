# esa-observability

Docker Compose observability stack for the ESA projects, built on the
VictoriaMetrics stack with Grafana as the UI. It accepts OpenTelemetry
**traces**, **metrics**, and **logs** over OTLP and stores each signal in an
OTLP-native VictoriaMetrics backend.

Primary consumer: [`../esa-dono-ui`](../esa-dono-ui).

## Architecture

```
                    ┌──────────────────────────────┐
 app (esa-dono-ui)  │  OpenTelemetry Collector      │
   ── OTLP ───────► │  otel/opentelemetry-collector │
   gRPC :4317       │  (contrib)                    │
   HTTP :4318       └───────┬──────────┬────────────┘
                           │          │            │
              metrics OTLP │   logs OTLP │  traces OTLP │
                           ▼          ▼            ▼
                  VictoriaMetrics  VictoriaLogs  VictoriaTraces
                     :8428           :9428         :10428
                           └──────┬───┴──────────────┘
                                  ▼
                             Grafana :3000
```

| Service           | Image                                   | Port  | Role                          |
| ----------------- | --------------------------------------- | ----- | ----------------------------- |
| `otelcol`         | `otel/opentelemetry-collector-contrib`  | 4317  | OTLP gRPC gateway             |
|                   |                                         | 4318  | OTLP HTTP gateway             |
| `victoriametrics` | `victoriametrics/victoria-metrics`      | 8428  | Metrics storage               |
| `victorialogs`    | `victoriametrics/victoria-logs`         | 9428  | Logs storage                  |
| `victoriatraces`  | `victoriametrics/victoria-traces`       | 10428 | Traces storage                |
| `grafana`         | `grafana/grafana`                       | 3000  | Dashboards / exploration      |

## Quick start

```sh
cp .env.example .env     # optional: customize ports / Grafana creds
docker compose up -d
```

- Grafana: http://localhost:3000 (`admin` / `admin`, see `.env`)
- VictoriaMetrics UI: http://localhost:8428
- VictoriaLogs UI: http://localhost:9428/select/vmui/

Grafana is provisioned with three data sources: **VictoriaMetrics** (default),
**VictoriaLogs**, and **VictoriaTraces**.

## Sending telemetry

Point any OTel SDK / collector at the gateway:

| Signal  | Protocol | Endpoint              |
| ------- | -------- | --------------------- |
| traces  | gRPC     | `localhost:4317`      |
| traces  | HTTP     | `http://localhost:4318/v1/traces` |
| metrics | gRPC     | `localhost:4317`      |
| metrics | HTTP     | `http://localhost:4318/v1/metrics` |
| logs    | gRPC     | `localhost:4317`      |
| logs    | HTTP     | `http://localhost:4318/v1/logs` |

## Wiring up esa-dono-ui

`esa-dono-ui` currently only exposes Prometheus metrics at `/api/metrics`
(gated by `METRICS_API_KEY`). To emit full OTLP telemetry it needs to:

1. Add the `@opentelemetry/*` SDK packages (tracing, metrics, logs) plus
   auto-instrumentation for its HTTP server and the frontend.
2. Configure the exporters to target the collector above
   (`OTEL_EXPORTER_OTLP_ENDPOINT=http://<host>:4318`).
3. Use the same `service.name` resource attribute across backend and frontend
   so traces from both are linked.

Add the gateway to the sibling's `docker-compose.yml` as an external target:

```yaml
# esa-dono-ui/docker-compose.yml (illustrative)
environment:
  - OTEL_EXPORTER_OTLP_ENDPOINT=http://otelcol:4318
  - OTEL_SERVICE_NAME=dono-backend
networks:
  default:
    name: esa-observability_default
    external: true
```

Alternatively, to keep the existing Prometheus metrics flowing, run `vmagent`
(or add a Prometheus scrape target) against `/api/metrics` with the
`METRICS_API_KEY` header.

## Storage

Data persists in named Docker volumes (`vm-data`, `vl-data`, `vt-data`,
`grafana-data`). Metrics retention is set to 3 months via
`-retentionPeriod=3` on VictoriaMetrics; tune in `compose.yaml`.

## Notes

- VictoriaTraces stores `service.name` (resource attribute) and span `name` as
  stream fields automatically — keep `service.name` stable and descriptive.
- Grafana Traces Drilldown / TraceQL is available by adding a **Tempo** data
  source pointed at VictoriaTraces instead of the Jaeger one.

## References

- [OpenTelemetry + VictoriaMetrics guide](https://docs.victoriametrics.com/guides/getting-started-with-opentelemetry/)
- [VictoriaMetrics/ai-observability demo](https://github.com/VictoriaMetrics/ai-observability)
