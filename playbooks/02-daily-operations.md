# Playbook 02 — Daily Operations

Start, stop, and watch the stack. This is the morning-routine page: lifecycle commands, the port map,
health checks, logs, and a quick tour of the dashboards. Deep observability (Grafana, Prometheus, Tempo,
Loki) lives in [08 — Observability](08-observability.md).

All commands run from the `BookieBreaker/` workspace root.

---

## 1. Stack lifecycle

```bash
task up      # start everything (docker compose up -d)
task down    # stop everything; data volumes survive
task build   # rebuild images after pulling service changes
task logs    # follow all logs
```

- `task down` keeps all data (Postgres, models, Ollama, metrics). Migrations and seeds do NOT need to be
  re-run after a normal down/up cycle.
- Re-run `task db:migrate` only after pulling service changes that include new migrations, and
  `task db:seed` only if you want fixture data restored.
- `task db:reset` is **destructive** — see [07 §7](07-troubleshooting.md#7-database-recovery).
- After editing `bookie-breaker-infra-ops/.env`, apply with `task down && task up`.

---

## 2. Service and port reference

| Service            | Port        | Health / entry point                                                                          |
| ------------------ | ----------- | --------------------------------------------------------------------------------------------- |
| lines-service      | 8001        | `/api/v1/lines/health`                                                                        |
| statistics-service | 8002        | `/api/v1/stats/health`                                                                        |
| simulation-engine  | 8003        | `/api/v1/sim/health`                                                                          |
| prediction-engine  | 8004        | `/api/v1/predict/health`                                                                      |
| bookie-emulator    | 8005        | `/api/v1/emulator/health`                                                                     |
| agent              | 8006        | `/api/v1/agent/health`                                                                        |
| mcp-server         | 8007        | `/health`                                                                                     |
| sharp-stub         | 8010        | opt-in via compose profile `stubs` — see [04 §3](04-parlays-props-and-live.md#3-live-betting) |
| ui                 | 3000        | `/healthz`                                                                                    |
| ollama             | 11434       | LLM backend (`ollama ls`)                                                                     |
| grafana            | 3001        | `admin` / `admin`; anonymous Viewer enabled                                                   |
| prometheus         | 9090        | metrics store                                                                                 |
| tempo              | 3200        | traces (7-day retention)                                                                      |
| loki               | 3100        | logs (30-day retention)                                                                       |
| otel-collector     | 4317 / 4318 | OTLP gRPC / HTTP ingest                                                                       |
| postgres           | 5432        | db `bookiebreaker`, user `bookiebreaker` / `$POSTGRES_PASSWORD` (default `localdev`)          |
| redis              | 6379        | cache + pub/sub, no persistence                                                               |

Python FastAPI services also serve interactive OpenAPI docs at `:80xx/docs` (8003–8007).

---

## 3. Health checks

```bash
bb health
```

Probes agent, lines-service, statistics-service, bookie-emulator, and prediction-engine concurrently.
Exit code 0 = all healthy; any failure exits 1 with the per-service detail. For services `bb health` does
not cover, hit the health paths in the table above, or:

```bash
docker compose -f bookie-breaker-infra-ops/docker-compose.yml ps   # container-level health column
```

---

## 4. Logs

```bash
task logs                             # everything, follow mode
task logs:service -- agent            # one service (compose service name)
task logs:service -- lines-service
```

For searching across time or correlating with traces, use Grafana Explore → Loki
([08 §5](08-observability.md#5-logs-in-loki)). Logs are retained for 30 days.

---

## 5. Dashboard quick glance

Web UI at <http://localhost:3000>:

| Page                    | Shows                                                                 |
| ----------------------- | --------------------------------------------------------------------- |
| `/`                     | overview: bankroll, active edges, recent runs                         |
| `/slate`                | today's games with predictions and best edge per game                 |
| `/edges`, `/edges/[id]` | current +EV edges and per-edge detail                                 |
| `/lines`                | current lines and movement                                            |
| `/live`, `/parlay`      | in-game edges and parlay builder ([04](04-parlays-props-and-live.md)) |
| `/performance`          | ROI, win rate, CLV over time                                          |
| `/bets`, `/bets/[id]`   | paper-bet ledger and detail                                           |

The UI updates live via server-sent events bridged from Redis pub/sub; if `REDIS_URL` is unset it
degrades to heartbeats only. Grafana system health is at <http://localhost:3001>
(dashboard **BookieBreaker - System Health**).

---

## 6. Hot-reload development

To iterate on one service, stop its container and run it from source with hot reload:

```bash
docker compose -f bookie-breaker-infra-ops/docker-compose.yml stop agent
task agent:dev        # uvicorn --reload on :8006
```

Equivalent tasks exist for every service: `lines-service:dev`, `statistics-service:dev` (air),
`simulation-engine:dev`, `prediction-engine:dev`, `bookie-emulator:dev`, `mcp-server:dev` (uvicorn),
`ui:dev` (pnpm), `cli:dev`. Full workflow detail: [operations/dev-workflow.md](../operations/dev-workflow.md).
