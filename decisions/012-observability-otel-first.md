# ADR-012: OpenTelemetry-First Observability

## Status
Accepted

## Context
The initial monitoring plan used Prometheus client libraries for direct metric exposition and structured JSON logging to stdout, with Loki as an optional add-on. This approach has several limitations:
- No distributed tracing — tracing cross-service requests relied on manual `X-Request-ID` propagation with no visualization
- Metrics and logs were disconnected — no correlation between a slow request in Prometheus and its log output in Loki
- Each language required different instrumentation libraries (prometheus/client_golang, prometheus-fastapi-instrumentator, prom-client)
- Observability was treated as optional/deferred — the base Docker Compose excluded monitoring infrastructure
- Retrofitting tracing into services after they're built is significantly harder than including it from the start

Complete observability (logs, metrics, traces — all correlated) is critical for debugging a distributed system with 10 services communicating via REST and Redis pub/sub.

## Decision

### OpenTelemetry as the Universal Instrumentation Standard

All deployed BookieBreaker components export telemetry via OpenTelemetry Protocol (OTLP) from Phase 1. This is mandatory, not optional.

### Architecture

```
Services (OTEL SDK) → OTLP → otel-collector → Prometheus (metrics)
                                              → Tempo (traces)
                                              → Loki (logs)
                                              → Grafana (visualization)
```

### Key Principles

1. **Single instrumentation API per language** — OTEL SDK replaces Prometheus client libs, custom log shippers, and manual trace propagation
2. **Collector as the routing layer** — services don't need to know about backends; otel-collector routes signals to the right place
3. **Correlation by default** — trace IDs appear in logs, link to traces in Tempo, and correlate with metrics in Prometheus
4. **Observability in base Docker Compose** — not an optional overlay. Every `docker compose up` includes the full stack
5. **All three signals from day one** — traces, metrics, and logs from the first service deployed

### Per-Language OTEL SDKs

| Language | SDK | Auto-instrumentation |
|---|---|---|
| Go | `go.opentelemetry.io/otel` | `otelecho` (HTTP), `otelpgx` (Postgres), `otelredis` (Redis) |
| Python | `opentelemetry-sdk` | `opentelemetry-instrumentation-fastapi`, `opentelemetry-instrumentation-httpx`, `opentelemetry-instrumentation-sqlalchemy` |
| TypeScript | `@opentelemetry/sdk-node` | `@opentelemetry/auto-instrumentations-node` |

### Grafana Stack

| Backend | Signal | Retention |
|---|---|---|
| Prometheus | Metrics | 30 days |
| Tempo | Traces | 7 days |
| Loki | Logs | 30 days |

All three data sources are configured in Grafana with cross-signal navigation (click a trace → see logs; click a metric spike → see traces from that window).

## Consequences

### Positive
- Distributed tracing from day one — can visualize a request flowing through statistics-service → simulation-engine → prediction-engine → agent
- Single instrumentation pattern across all languages (OTEL SDK)
- Logs, metrics, and traces are correlated by trace ID
- otel-collector provides a single configuration point for telemetry routing
- No service-level Prometheus endpoints needed — metrics flow through OTLP

### Negative
- Heavier base Docker Compose (~4 additional containers: otel-collector, Tempo, Loki, Grafana + Prometheus)
- Slightly longer `docker compose up` cold start time
- OTEL SDK adds a dependency to every service
- Learning curve for OTEL API/SDK across Go, Python, and TypeScript

### Neutral
- Prometheus still stores metrics — OTEL just changes how metrics get there (OTLP instead of scraping `/metrics`)
- Grafana is already planned — this decision adds Tempo and Loki as additional data sources
- The otel-collector is a CNCF graduated project — mature and well-supported
