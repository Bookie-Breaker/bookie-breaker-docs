# Playbook 08 — Observability

Working the telemetry stack: Grafana dashboards, Prometheus metrics, Tempo traces, and Loki logs. Design
rationale lives in [operations/monitoring-observability.md](../operations/monitoring-observability.md);
this is the hands-on tour.

---

## 1. Architecture in one paragraph

Every service exports OTLP telemetry to the **otel-collector** (gRPC :4317 / HTTP :4318), which fans out
by signal: traces → **Tempo** (:3200, 7-day retention), metrics → a Prometheus scrape endpoint on the
collector (:8889) scraped by **Prometheus** (:9090, 30-day retention), logs → **Loki** (:3100, 30-day
retention). **Grafana** (:3001) fronts all three with pre-provisioned datasources. Prometheus scrapes only
the collector — services are not scraped directly.

---

## 2. Grafana

<http://localhost:3001> — `admin` / `admin`; anonymous access is enabled with the Viewer role, so
read-only browsing needs no login.

Provisioned out of the box:

- Datasources: **Prometheus** (default), **Tempo**, **Loki** — pre-wired for cross-navigation (trace →
  logs, logs → trace, service graph).
- Dashboard: **BookieBreaker - System Health** (folder _BookieBreaker_) — service status, HTTP request
  rate, p95 latency, 4xx/5xx error rate, Redis memory.

That is the only provisioned dashboard; anything else you build lives in Grafana's own storage
(`grafana-data` volume). Export new dashboards to
`bookie-breaker-infra-ops/grafana/dashboards/` if they should survive a wipe and provision for everyone.

---

## 3. Metrics in Prometheus

Grafana → Explore → Prometheus (or raw Prometheus at <http://localhost:9090>). Useful starting queries:

```promql
sum by (service_name) (rate(http_server_duration_milliseconds_count[5m]))   # request rate per service
histogram_quantile(0.95, sum by (le, service_name) (rate(http_server_duration_milliseconds_bucket[5m])))
sum by (service_name) (rate(http_server_duration_milliseconds_count{http_status_code=~"5.."}[5m]))
```

Metric names follow OTel HTTP semantic conventions; browse what actually exists via Prometheus's
`/graph` autocomplete or `curl -s localhost:8889/metrics | grep -o '^[a-z_]*' | sort -u` against the
collector's scrape endpoint.

---

## 4. Traces in Tempo

Grafana → Explore → Tempo. To trace a pipeline run end-to-end:

1. Trigger a run (`bb pipeline run --league NBA`) and note the time.
2. Search service `agent`, limit to the last 15 minutes; open the span for
   `POST /api/v1/agent/pipeline/run` or the background run span.
3. The trace fans out through simulation-engine → prediction-engine → lines-service → bookie-emulator;
   long or errored spans show where a run spends its time.
4. From any span, jump to correlated logs (trace-to-logs is pre-wired to Loki).

Traces are retained for **7 days** (shorter than logs/metrics) — investigate while fresh.

---

## 5. Logs in Loki

Grafana → Explore → Loki. Logs arrive via OTLP with service attribution:

```logql
{service_name="agent"}                                   # one service's logs
{service_name="agent"} |= "pipeline"                     # substring filter
{service_name=~"lines-service|statistics-service"} |= "error"
```

Log lines carry `trace_id`; Grafana links them to the Tempo trace automatically (derived field). This is
the searchable alternative to `task logs:service`, and it spans the full 30-day retention.

---

## 6. Alerting

**No alert rules are provisioned** — Prometheus has no rules files and no Alertmanager, and Grafana ships
without alert definitions. Operational alerting today is application-level: the agent publishes
`events:edge.detected` to Redis and persists rows to `agent.edge_alerts`, surfaced via
`GET /api/v1/agent/alerts` and the UI. If you add infrastructure alerts, provision them under
`bookie-breaker-infra-ops/grafana/` or `prometheus/` so they are reproducible.

---

## 7. Health endpoint reference

For scripted checks (all return 200 when healthy):

| Service            | URL                                            |
| ------------------ | ---------------------------------------------- |
| lines-service      | `http://localhost:8001/api/v1/lines/health`    |
| statistics-service | `http://localhost:8002/api/v1/stats/health`    |
| simulation-engine  | `http://localhost:8003/api/v1/sim/health`      |
| prediction-engine  | `http://localhost:8004/api/v1/predict/health`  |
| bookie-emulator    | `http://localhost:8005/api/v1/emulator/health` |
| agent              | `http://localhost:8006/api/v1/agent/health`    |
| mcp-server         | `http://localhost:8007/health`                 |
| ui                 | `http://localhost:3000/healthz`                |

`bb health` covers the five the CLI talks to; the compose healthchecks probe all of them continuously.
