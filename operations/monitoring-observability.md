# Monitoring & Observability

How BookieBreaker services emit logs, metrics, traces, and health signals. Built on **OpenTelemetry (OTEL) as the universal instrumentation standard** from Phase 1. All deployed components export OTEL telemetry — this is not optional. See [ADR-012](../decisions/012-observability-otel-first.md).

---

## 0. OpenTelemetry Architecture

All services instrument with OpenTelemetry SDKs and export via OTLP to a central otel-collector running in Docker Compose. The collector routes signals to their backends:

- **Traces** → Grafana Tempo (distributed tracing across services)
- **Metrics** → Prometheus (via OTLP receiver, replacing direct Prometheus client instrumentation)
- **Logs** → Grafana Loki (structured log aggregation)

All three signals are visualized in Grafana with cross-signal correlation (trace ID links logs to traces to metrics).

### OTEL SDK Libraries

| Language | SDK | Auto-instrumentation |
|----------|-----|---------------------|
| Go | `go.opentelemetry.io/otel` | `otelecho` middleware for HTTP, `otelpgx` for Postgres, `otelredis` for Redis |
| Python | `opentelemetry-sdk` | `opentelemetry-instrumentation-fastapi`, `opentelemetry-instrumentation-httpx`, `opentelemetry-instrumentation-sqlalchemy` |
| TypeScript | `@opentelemetry/sdk-node` | `@opentelemetry/auto-instrumentations-node` |

### otel-collector Configuration

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  prometheus:
    endpoint: 0.0.0.0:8889
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

processors:
  batch:
    timeout: 5s
    send_batch_size: 1024

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/tempo]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [loki]
```

### Docker Compose Observability Stack

```yaml
# docker-compose.yml (included in base, not optional overlay)
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.100.0
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8889:8889"   # Prometheus metrics endpoint
    volumes:
      - ./otel-collector-config.yaml:/etc/otelcol-contrib/config.yaml

  tempo:
    image: grafana/tempo:2.4
    ports:
      - "3200:3200"
    volumes:
      - tempo-data:/var/tempo

  loki:
    image: grafana/loki:3.0
    ports:
      - "3100:3100"
    volumes:
      - loki-data:/loki

  prometheus:
    image: prom/prometheus:v2.53
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus

  grafana:
    image: grafana/grafana:11.0
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
      - grafana-data:/var/lib/grafana
```

**Note:** The observability stack is part of the base `docker-compose.yml`, not an optional overlay. All services must have observability from the start.

---

## 1. Logging

### Structured JSON Logging

All services emit structured JSON logs to stdout. Logs are also forwarded to the otel-collector via OTLP for centralized querying in Loki/Grafana.

**Standard log fields (all services):**

```json
{
  "timestamp": "2026-03-30T14:22:00.123Z",
  "level": "info",
  "service": "lines-service",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "message": "Lines ingested successfully",
  "league": "NBA",
  "count": 42,
  "duration_ms": 312
}
```

```json
{
  "timestamp": "2026-03-30T14:22:01.456Z",
  "level": "error",
  "service": "simulation-engine",
  "request_id": "550e8400-e29b-41d4-a716-446655440001",
  "message": "Simulation failed for game",
  "error": {
    "type": "ValueError",
    "message": "Insufficient team data for simulation",
    "stack": "..."
  },
  "game_id": "game-abc",
  "league": "NFL"
}
```

**Required fields:** `timestamp`, `level`, `service`, `message`.
**Recommended fields:** `request_id`, `error` (on errors), `duration_ms` (on request completion).
**Optional context fields:** Any domain-relevant key-value pairs (league, game_id, batch_id, etc.).

**Log levels:** `debug`, `info`, `warn`, `error`. Default level is `info` in production, `debug` in development. Controlled via `LOG_LEVEL` environment variable per service.

### Correlation IDs & Distributed Tracing

OTEL trace context (`traceparent` header) is the primary correlation mechanism — it propagates automatically through OTEL-instrumented HTTP clients and servers. The legacy `X-Request-ID` header is also generated for backward compatibility and included in log output. Both trace ID and request ID appear in structured log fields, enabling Grafana to correlate logs ↔ traces ↔ metrics.

**Implementation:**

- **Go (Echo middleware):** Extract `X-Request-ID` from incoming request. If absent, generate a UUID v4. Set it on the response and inject into the request context. All log statements include the request ID from context.
- **Python (FastAPI middleware):** Same pattern using a custom middleware that reads/generates `X-Request-ID` and stores it in a context variable accessible to `structlog`.
- **Redis pub/sub events:** Include the `correlation_id` field in the event envelope (defined in [communication-patterns.md](../architecture/communication-patterns.md)) to link events back to the originating request.

### Per-Language Libraries

| Language | Library | Notes |
|----------|---------|-------|
| Go | `log/slog` (stdlib, Go 1.21+) | Structured logger in the standard library. Use `slog.JSONHandler` for JSON output. |
| Python | `structlog` | Processors for JSON rendering, context binding, and stdlib integration. Configure once at app startup. |
| TypeScript | `pino` | Fast JSON logger. Used in SvelteKit server hooks for SSR request logging. |

### Log Aggregation

**Primary:** Logs are exported via OTLP to the otel-collector, which forwards them to Loki. Query logs in Grafana with full trace ID correlation — click a trace in Tempo to see associated logs.

**Fallback:** `docker compose logs -f` with optional service filtering still works for quick debugging during development.

---

## 2. Metrics

### OTEL Metrics → Prometheus

Services emit metrics via OTEL SDK (not direct Prometheus client libraries). The otel-collector exports metrics to Prometheus in exposition format. Prometheus scrapes the otel-collector's metrics endpoint. This means services don't need a `/metrics` endpoint — instrumentation is handled entirely through OTLP export.

### Per-Service Key Metrics

#### lines-service (Go)

| Metric | Type | Description |
|--------|------|-------------|
| `lines_ingestion_latency_seconds` | Histogram | Time to complete a full ingestion cycle from an external source |
| `lines_ingested_total` | Counter | Total lines ingested, labeled by `league` and `source` |
| `lines_api_errors_total` | Counter | External API errors, labeled by `source` and `error_type` |
| `lines_active_games_gauge` | Gauge | Number of games with active (non-final) lines |
| `lines_source_last_success_timestamp` | Gauge | Unix timestamp of last successful fetch per source |

#### statistics-service (Go)

| Metric | Type | Description |
|--------|------|-------------|
| `stats_cache_hit_total` | Counter | Redis cache hits, labeled by `league` and `data_type` |
| `stats_cache_miss_total` | Counter | Redis cache misses, labeled by `league` and `data_type` |
| `stats_cache_hit_rate` | Gauge | Rolling cache hit ratio (computed from counters) |
| `stats_api_latency_seconds` | Histogram | External stats API call latency, labeled by `source` |
| `stats_api_errors_total` | Counter | External stats API errors, labeled by `source` |

#### simulation-engine (Python)

| Metric | Type | Description |
|--------|------|-------------|
| `simulation_duration_seconds` | Histogram | Wall-clock time per simulation batch |
| `simulations_total` | Counter | Total simulations completed, labeled by `league` |
| `simulation_iterations` | Histogram | Number of iterations per simulation run |
| `simulation_convergence_rate` | Gauge | Fraction of simulations that converged within tolerance |
| `simulation_errors_total` | Counter | Simulation failures, labeled by `error_type` |

#### prediction-engine (Python)

| Metric | Type | Description |
|--------|------|-------------|
| `prediction_latency_seconds` | Histogram | Time to generate predictions for a batch |
| `prediction_model_accuracy_rolling` | Gauge | Rolling accuracy over last N predictions, labeled by `league` and `bet_type` |
| `prediction_calibration_error` | Gauge | Mean calibration error (predicted vs. actual probability) |
| `prediction_edge_magnitude` | Histogram | Distribution of detected edge sizes |
| `predictions_total` | Counter | Total predictions generated |

#### bookie-emulator (Python)

| Metric | Type | Description |
|--------|------|-------------|
| `bookie_bets_placed_total` | Counter | Paper bets placed, labeled by `league` and `bet_type` |
| `bookie_bets_graded_total` | Counter | Paper bets graded, labeled by `result` (win/loss/push) |
| `bookie_roi_rolling` | Gauge | Rolling ROI over configurable window |
| `bookie_clv_average` | Gauge | Average Closing Line Value across recent bets |
| `bookie_bankroll_current` | Gauge | Current paper bankroll value |
| `bookie_drawdown_current` | Gauge | Current drawdown from peak bankroll |

#### agent (Python)

| Metric | Type | Description |
|--------|------|-------------|
| `agent_pipeline_duration_seconds` | Histogram | End-to-end pipeline run duration |
| `agent_edges_detected_total` | Counter | Edges detected, labeled by `league` and `bet_type` |
| `agent_analysis_latency_seconds` | Histogram | LLM analysis call latency |
| `agent_llm_tokens_total` | Counter | LLM tokens consumed, labeled by `direction` (input/output) |
| `agent_pipeline_errors_total` | Counter | Pipeline failures, labeled by `failed_step` |
| `agent_llm_provider` | Gauge | Current LLM provider in use (label: `provider`=anthropic\|ollama) |

#### Local LLM / Ollama (when running)

| Metric | Type | Description |
|--------|------|-------------|
| `ollama_model_load_seconds` | Histogram | Time to load a model into memory |
| `ollama_tokens_per_second` | Gauge | Current inference throughput |
| `ollama_vram_usage_bytes` | Gauge | GPU memory used by loaded models |

### Per-Language Instrumentation Libraries

| Language | Library | Notes |
|----------|---------|-------|
| Go | `go.opentelemetry.io/otel` + `otelecho` | Auto-instruments Echo HTTP routes. Custom metrics via OTEL meter API. Exports via OTLP gRPC to otel-collector. |
| Python | `opentelemetry-sdk` + `opentelemetry-instrumentation-fastapi` | Auto-instruments FastAPI routes. Custom metrics via OTEL meter API. Exports via OTLP gRPC to otel-collector. |
| TypeScript | `@opentelemetry/sdk-node` + auto-instrumentations | Auto-instruments SvelteKit server. Custom metrics via OTEL meter API. Exports via OTLP gRPC to otel-collector. |

### Prometheus Configuration

Prometheus scrapes metrics from the otel-collector's Prometheus exporter endpoint, not from individual services. This simplifies the Prometheus config to a single scrape target:

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "otel-collector"
    static_configs:
      - targets: ["otel-collector:8889"]
```

---

## 3. Dashboards

### Grafana

Grafana runs alongside the application services in Docker Compose. Dashboard JSON definitions are version-controlled in `bookie-breaker-infra-ops/grafana/dashboards/` and provisioned automatically on startup.

### Dashboard Layouts

#### System Health Dashboard

Overview of all services at a glance.

- **Service status grid:** Up/down status for each service (derived from health check metrics or Prometheus `up` metric)
- **Request rate and error rate:** Per-service HTTP request rates and 5xx error rates
- **Response latency (p50, p95, p99):** Per-service response time percentiles
- **Redis connection pool:** Active connections, command rate, memory usage
- **PostgreSQL connections:** Active connections per database, query latency

#### Prediction Pipeline Dashboard

End-to-end visibility into a pipeline run.

- **Pipeline run timeline:** Gantt-style visualization of pipeline step durations (ingestion, simulation, prediction, edge detection)
- **Simulation metrics:** Duration histogram, convergence rate, iterations distribution
- **Prediction metrics:** Model accuracy trend, calibration error over time, edge magnitude distribution
- **LLM usage:** Token consumption rate, analysis latency, error rate

#### Paper Trading Performance Dashboard

The most important dashboard -- validates whether the system works.

- **ROI over time:** Rolling ROI curve with confidence intervals
- **Win rate by league and bet type:** Heatmap of win rates across categories
- **CLV distribution:** Histogram of Closing Line Value across all bets
- **Bankroll curve:** Paper bankroll value over time with drawdown overlay
- **Calibration plot:** Predicted probability vs. actual win rate (reliability diagram)
- **Recent bets table:** Last 20 bets with placement odds, closing odds, result, and CLV

#### Data Source Health Dashboard

Monitors external API reliability.

- **Source uptime:** Last successful fetch timestamp per source, with staleness alerts
- **Ingestion rate:** Lines ingested per minute, stats fetched per hour
- **API error rates:** Error counts per external source, with error type breakdown
- **Cache performance:** Hit rate over time, miss rate spikes indicating cache invalidation or cold starts

### Grafana Provisioning

Grafana is provisioned with data sources for all three backends (Prometheus, Tempo, Loki) and pre-built dashboards. See the Docker Compose observability stack in Section 0 above for container configuration.

Grafana data sources are configured via provisioning YAML:

```yaml
# grafana/provisioning/datasources/datasources.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
  - name: Tempo
    type: tempo
    url: http://tempo:3200
  - name: Loki
    type: loki
    url: http://loki:3100
```

---

## 4. Alerting

### Alert Rules

Alerts are defined in Prometheus alerting rules and evaluated by Prometheus. Grafana can also evaluate alerts directly if Prometheus Alertmanager is not deployed.

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| Data source down | `lines_source_last_success_timestamp` older than 30 minutes | Warning | Check external API status, verify API key validity |
| Prediction pipeline failed | `agent_pipeline_errors_total` increases | Critical | Check agent logs, identify failed step, verify downstream service health |
| Unusual edge volume | `agent_edges_detected_total` rate > 3x rolling average | Warning | Review edges for data quality issues (garbage in, garbage out) |
| Paper trading drawdown | `bookie_drawdown_current` > configurable threshold (e.g., 20%) | Warning | Review recent predictions, check for model drift or data issues |
| High simulation failure rate | `simulation_errors_total` rate > 10% of `simulations_total` | Warning | Check statistics-service data availability, review simulation parameters |
| Service unhealthy | Health check endpoint returns non-200 for > 2 minutes | Critical | Check service logs, restart container if needed |
| Redis memory high | Redis used memory > 80% of max | Warning | Review cache TTLs, check for key leaks |

### Alert Channels

**Phase 1 (solo dev):** Alerts surface in Grafana's built-in alert UI. Check the Grafana dashboard periodically. Log-level alerts are always visible in `docker compose logs`.

**Phase 2 (optional):** Integrate Grafana with a Discord or Slack webhook for push notifications on critical alerts. Configuration is a single webhook URL in Grafana's notification channels.

```yaml
# Grafana contact point configuration (added via UI or provisioning)
# Discord webhook example:
# URL: https://discord.com/api/webhooks/{id}/{token}
# Slack webhook example:
# URL: https://hooks.slack.com/services/T.../B.../...
```

---

## 5. Health Checks

### Health Endpoint Specification

Every service exposes `GET /health` (no authentication required). This endpoint is used by Docker Compose health checks, Prometheus service discovery, and the system health dashboard.

**Response format:**

```json
{
  "status": "healthy",
  "service": "lines-service",
  "version": "1.2.3",
  "uptime_seconds": 86421,
  "dependencies": {
    "postgres": {
      "status": "healthy",
      "latency_ms": 2
    },
    "redis": {
      "status": "healthy",
      "latency_ms": 1
    },
    "odds_api": {
      "status": "healthy",
      "last_success": "2026-03-30T14:20:00Z"
    }
  }
}
```

**Status values:**
- `healthy` -- All dependencies reachable, service functioning normally
- `degraded` -- Service is running but one or more non-critical dependencies are down (e.g., cache miss fallback active)
- `unhealthy` -- Service cannot fulfill its primary function

**HTTP status codes:**
- `200` for `healthy` or `degraded`
- `503` for `unhealthy`

### Docker Compose Health Checks

Health checks enable dependency ordering. Services that depend on databases or Redis wait for those to be healthy before starting.

```yaml
# Example health check configuration in docker-compose.yml
services:
  lines-service:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8001/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
    depends_on:
      redis:
        condition: service_healthy
      lines-db:
        condition: service_healthy

  redis:
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

  lines-db:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 3s
      retries: 3
```

### Health Check Implementation Notes

- **Go services:** A lightweight handler that pings Postgres (`db.PingContext`) and Redis (`client.Ping`) and returns aggregate status. Keep it fast -- health checks run every 30 seconds.
- **Python services:** FastAPI endpoint that performs the same dependency pings. Use `asyncio.wait_for` with a 2-second timeout on each dependency check to prevent health checks from hanging.
- **External API health:** Do not call external APIs in the health check (too slow, rate limits). Instead, report the timestamp of the last successful external fetch. Mark as degraded if the last success is older than the expected fetch interval.

---

## Observability Stack Summary

| Component | Tool | Purpose | Required? |
|-----------|------|---------|-----------|
| Instrumentation | OpenTelemetry SDK (per-language) | Unified traces, metrics, logs export via OTLP | Yes (all services) |
| Collector | otel-collector | Receives OTLP, routes to backends | Yes |
| Logging | slog / structlog / pino | Structured JSON logs to stdout + OTLP | Yes |
| Log aggregation | Grafana Loki | Centralized log querying | Yes |
| Traces | Grafana Tempo | Distributed tracing | Yes |
| Metrics | Prometheus (via otel-collector) | Metric collection and storage | Yes |
| Dashboards | Grafana | Unified visualization for traces, metrics, logs | Yes |
| Alerting | Grafana alerts | Push notifications on critical conditions | Optional (Grafana UI is sufficient initially) |
| Health checks | /health endpoints | Service status and dependency ordering | Yes |

### Deployment

The observability stack is part of the base `docker-compose.yml` — **not an optional overlay**. All services must have observability from day one. This is a deliberate decision: debugging distributed systems without tracing is painful, and retrofitting instrumentation is harder than including it from the start.

```bash
# All services + full observability (default)
docker compose up

# Optional GPU overlay for local LLM
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up
```
