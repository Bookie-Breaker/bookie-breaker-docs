# Monitoring & Observability

How BookieBreaker services emit logs, metrics, and health signals. Designed for a solo developer running Docker Compose locally, with a clear upgrade path to production-grade observability.

---

## 1. Logging

### Structured JSON Logging

All services emit structured JSON logs to stdout. Docker captures these automatically, and they can be aggregated downstream without custom parsers.

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

### Correlation IDs

Every inbound HTTP request generates or propagates a correlation ID via the `X-Request-ID` header. When service A calls service B, it forwards the same `X-Request-ID`. This allows tracing a single user request across all services it touches.

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

**Development (default):** `docker compose logs -f` with optional service filtering. Sufficient for debugging during development.

**Production-ready (optional):** Loki + Promtail for centralized log aggregation. Promtail runs as a sidecar or Docker log driver, ships logs to Loki. Loki integrates with Grafana for querying. This is lightweight compared to the ELK stack and shares the Grafana UI already used for metrics.

```yaml
# docker-compose.observability.yml (optional overlay)
services:
  loki:
    image: grafana/loki:3.0
    ports:
      - "3100:3100"
    volumes:
      - loki-data:/loki

  promtail:
    image: grafana/promtail:3.0
    volumes:
      - /var/log:/var/log
      - /var/run/docker.sock:/var/run/docker.sock
    command: -config.file=/etc/promtail/config.yml
```

---

## 2. Metrics

### Prometheus Exposition

Every service exposes a `/metrics` endpoint in Prometheus exposition format. Prometheus scrapes these endpoints on a configurable interval (default: 15 seconds).

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

### Per-Language Instrumentation Libraries

| Language | Library | Notes |
|----------|---------|-------|
| Go | `prometheus/client_golang` | Register custom collectors. Echo middleware for automatic HTTP request metrics. |
| Python | `prometheus-fastapi-instrumentator` | Auto-instruments FastAPI routes (latency, status codes, request size). Add custom metrics via `prometheus_client`. |
| TypeScript | `prom-client` | Expose from a SvelteKit server endpoint at `/metrics`. |

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "lines-service"
    static_configs:
      - targets: ["lines-service:8001"]

  - job_name: "statistics-service"
    static_configs:
      - targets: ["statistics-service:8002"]

  - job_name: "simulation-engine"
    static_configs:
      - targets: ["simulation-engine:8003"]

  - job_name: "prediction-engine"
    static_configs:
      - targets: ["prediction-engine:8004"]

  - job_name: "bookie-emulator"
    static_configs:
      - targets: ["bookie-emulator:8005"]

  - job_name: "agent"
    static_configs:
      - targets: ["agent:8006"]

  - job_name: "ui"
    static_configs:
      - targets: ["ui:3000"]
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

```yaml
# docker-compose.observability.yml
services:
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

  prometheus:
    image: prom/prometheus:v2.53
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
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
| Logging | slog / structlog / pino | Structured JSON logs to stdout | Yes |
| Log aggregation | docker logs / Loki | Centralized log querying | docker logs required, Loki optional |
| Metrics | Prometheus | Metric collection and storage | Yes (for dashboards) |
| Dashboards | Grafana | Visualization and alerting | Yes |
| Alerting | Grafana alerts | Push notifications on critical conditions | Optional (Grafana UI is sufficient initially) |
| Health checks | /health endpoints | Service status and dependency ordering | Yes |

### Deployment

The observability stack runs as an optional Docker Compose overlay:

```bash
# Development (application services only)
docker compose up

# Development with observability
docker compose -f docker-compose.yml -f docker-compose.observability.yml up
```

This keeps the base `docker-compose.yml` lean for fast startup during development while making full observability available when needed.
