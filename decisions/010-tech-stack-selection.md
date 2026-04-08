# ADR-010: Tech Stack Selection

## Status

Accepted

## Context

Per ADR-006, we select the best tool per service while keeping the total language count manageable for a solo developer.
The selection is informed by:

- Statistics data sources are Python packages (nfl_data_py, nba_api, pybaseball) — forces Python for data-adjacent
  services
- ML/simulation libraries (NumPy, SciPy, scikit-learn, XGBoost) — forces Python for compute services
- Anthropic SDK and MCP SDK are most mature in Python — favors Python for agent/MCP
- Data retrieval services prioritize speed — favors Go
- CLI needs polished terminal output — Go's Charm ecosystem is best-in-class
- UI is a data-heavy dashboard — SvelteKit + ECharts for rich visualizations
- Infrastructure is configuration, not application code — shell/YAML

## Decision

### Language Split: Go + Python + TypeScript (3 languages)

| Service            | Language      | Framework         | Key Libraries                                                     |
| ------------------ | ------------- | ----------------- | ----------------------------------------------------------------- |
| lines-service      | Go 1.22+      | Echo              | pgx, go-redis, oapi-codegen                                       |
| statistics-service | Go 1.22+      | Echo              | go-redis, sport-specific API adapters                             |
| simulation-engine  | Python 3.12+  | FastAPI           | NumPy, SciPy, uvicorn                                             |
| prediction-engine  | Python 3.12+  | FastAPI           | scikit-learn, XGBoost, pandas, uvicorn                            |
| agent              | Python 3.12+  | FastAPI           | anthropic SDK, ollama SDK, httpx, pydantic                        |
| mcp-server         | Python 3.12+  | MCP SDK (FastMCP) | mcp, anthropic                                                    |
| bookie-emulator    | Python 3.12+  | FastAPI           | pandas (performance analysis), pydantic                           |
| cli                | Go 1.22+      | Cobra             | Bubble Tea, Lip Gloss, Glamour (Charm ecosystem)                  |
| ui                 | TypeScript 5+ | SvelteKit         | Apache ECharts, Skeleton UI, svelte-echarts                       |
| infra-ops          | Shell/YAML    | Taskfile          | Docker Compose, Kustomize, GitHub Actions, otel-collector, Ollama |

### Storage Architecture

A minimal, pragmatic storage topology:

| Store         | Technology                  | Purpose                                                                                   | Services Using                     |
| ------------- | --------------------------- | ----------------------------------------------------------------------------------------- | ---------------------------------- |
| Lines DB      | PostgreSQL 16 + TimescaleDB | Line snapshots with time-series optimization, history cap to manage storage               | lines-service (owner)              |
| App DB        | PostgreSQL 16               | Predictions, model versions, paper bets, grades, bankroll                                 | prediction-engine, bookie-emulator |
| Cache/Pub-Sub | Redis 7                     | Stats caching (generous TTL), simulation result caching, pub/sub event bus, session cache | All services                       |

**Key storage decisions:**

- **Single PostgreSQL instance** with TimescaleDB extension enabled, per-service schemas. TimescaleDB hypertables for
  lines-service; standard tables for everything else.
- **Statistics are NOT stored in a database.** Statistics-service is a cache + enrichment layer — fetches from external
  APIs, caches in Redis with generous TTL, computes derived features in-memory. External APIs are the source of truth.
- **Lines history cap** — TimescaleDB compression + retention policy to prevent unbounded storage growth. Keep granular
  data for current season, compress older data.
- **Simulation results cached in Redis** — ephemeral, regenerated as needed. Only the latest run per game matters.

### Observability Stack

| Component       | Technology                       | Purpose                                             |
| --------------- | -------------------------------- | --------------------------------------------------- |
| Instrumentation | OpenTelemetry SDK (per-language) | Unified traces, metrics, and logs export via OTLP   |
| Collector       | otel-collector                   | Receives OTLP from all services, routes to backends |
| Metrics         | Prometheus                       | Metric storage, queried by Grafana                  |
| Traces          | Grafana Tempo                    | Distributed trace storage and querying              |
| Logs            | Grafana Loki                     | Log aggregation and querying                        |
| Dashboards      | Grafana                          | Unified visualization for all three signals         |

All deployed components must export OTEL telemetry from Phase 1. See [ADR-012](./012-observability-otel-first.md).

### LLM Infrastructure

| Component   | Technology                  | Purpose                                                                             |
| ----------- | --------------------------- | ----------------------------------------------------------------------------------- |
| Cloud LLM   | Anthropic API               | Primary LLM for production-quality analysis (Sonnet for detail, Haiku for routine)  |
| Local LLM   | Ollama                      | Self-hosted models for development, cost-free experimentation, and offline use      |
| Abstraction | Provider interface in agent | Config-only switching between Anthropic and Ollama (`LLM_PROVIDER`, `LLM_BASE_URL`) |

See [ADR-011](./011-local-llm-strategy.md).

### Future Evaluation: kagent (CNCF Sandbox)

kagent is a Kubernetes-native AI agent framework with built-in chat UI and native MCP server support. Currently CNCF
Sandbox maturity (v0.7.14, Feb 2026). Evaluate during Phase 5 for potential adoption as K8s agent orchestration layer.
Key value: reduces custom UI/agent plumbing. Key risk: Sandbox status means API instability.

### Per-Language Standards

#### Go (lines-service, statistics-service, cli)

- **Package manager:** Go modules
- **Linting:** golangci-lint (with default preset)
- **Formatting:** gofmt (enforced)
- **Testing:** Standard library `testing` + testify for assertions
- **Structure:** Standard Go project layout (`cmd/`, `internal/`, `pkg/`)

#### Python (simulation-engine, prediction-engine, agent, mcp-server, bookie-emulator)

- **Package manager:** uv (fast, modern pip replacement)
- **Linting:** ruff (replaces flake8, isort, black)
- **Formatting:** ruff format
- **Type checking:** mypy or pyright (strict mode)
- **Testing:** pytest + pytest-asyncio
- **Structure:** src layout (`src/<package>/`)

#### TypeScript (ui)

- **Package manager:** pnpm
- **Linting:** ESLint + svelte plugin
- **Formatting:** Prettier
- **Testing:** Vitest + Playwright (e2e)
- **Structure:** SvelteKit default (`src/routes/`, `src/lib/`)

### Why Each Choice

**Go for data retrieval (lines-service, statistics-service):**
Speed is the primary factor. Go's HTTP client performance, goroutines for concurrent API polling, and low memory
footprint make it ideal for high-throughput data ingestion. Echo provides a clean REST framework with middleware
support.

**Go for CLI:**
The Charm ecosystem (Bubble Tea, Lip Gloss, Glamour) produces the most polished terminal UI in any language. Single
binary distribution means no runtime dependencies for users. Cobra provides best-in-class command structure,
auto-completion, and help generation.

**Python for compute/ML (simulation-engine, prediction-engine):**
No other language matches Python's ML/scientific computing ecosystem. NumPy, SciPy, scikit-learn, XGBoost, and pandas
are irreplaceable. FastAPI provides async REST with automatic OpenAPI spec generation.

**Python for agent/MCP:**
The Anthropic SDK and MCP SDK are most mature and best-documented in Python. The entire LLM tooling ecosystem
(LangChain, LangGraph, etc.) is Python-first. The agent works closely with the MCP server, so same language reduces
friction.

**Python for bookie-emulator:**
Paper trading performance analysis (ROI curves, calibration, Brier scores, Kelly optimization) benefits from pandas and
scipy.stats. FastAPI keeps it consistent with other Python services.

**SvelteKit + ECharts for UI:**
ECharts is the most feature-rich charting library for the data visualizations needed (candlestick-style line movement,
distribution histograms, calibration plots, time-series performance). SvelteKit's reactivity model is ideal for
live-updating dashboards. Skeleton UI provides dashboard-friendly components.

**Taskfile for infra-ops:**
Modern Makefile replacement with YAML syntax, cross-platform support, and built-in task dependencies. Perfect for
orchestrating multi-repo operations (build all, test all, start all, generate clients).

## Consequences

### Positive

- 3 languages keeps cognitive overhead manageable for a solo developer
- Each language serves a clear purpose: Go (speed/CLI), Python (ML/LLM), TypeScript (UI)
- Minimal storage footprint — Redis cache + one Postgres instance with TimescaleDB
- uv and pnpm are the fastest package managers in their ecosystems
- Statistics-service stays simple by treating external APIs as source of truth

### Negative

- 3 languages means 3 CI pipeline configurations, 3 linting setups, 3 test frameworks
- Go ↔ Python service communication requires OpenAPI codegen discipline (ADR-009)
- SvelteKit is less battle-tested than React for large dashboards (but growing fast)
- Redis as stats cache means cache misses hit external APIs (need graceful degradation)

### Neutral

- FastAPI auto-generates OpenAPI specs, which feeds directly into the codegen pipeline (ADR-009)
- Echo can also generate OpenAPI specs with middleware, keeping Go services spec-compatible
- All three languages have official Docker base images, simplifying containerization
