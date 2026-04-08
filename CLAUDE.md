# BookieBreaker — Shared Claude Configuration

## System Overview

BookieBreaker is a distributed sports prediction and paper trading system that identifies positive expected value (+EV) betting edges against sportsbooks using hybrid prediction (Monte Carlo simulation + ML calibration). It supports NFL, NBA, MLB, and NCAA leagues.

## Service Map

| Repo                              | Language             | Purpose                                                       | Port |
| --------------------------------- | -------------------- | ------------------------------------------------------------- | ---- |
| bookie-breaker-lines-service      | Go                   | Ingest and serve betting lines from external APIs             | 8001 |
| bookie-breaker-statistics-service | Go                   | Team/player stats, schedules, game results with Redis caching | 8002 |
| bookie-breaker-simulation-engine  | Python               | Monte Carlo simulations for outcome distributions             | 8003 |
| bookie-breaker-prediction-engine  | Python               | ML-based probability calibration and edge detection           | 8004 |
| bookie-breaker-bookie-emulator    | Python               | Paper trading and performance tracking                        | 8005 |
| bookie-breaker-agent              | Python               | LLM-powered orchestration, pipeline coordination              | 8006 |
| bookie-breaker-mcp-server         | Python               | MCP tool server for Claude integration                        | 8007 |
| bookie-breaker-cli                | Go                   | Terminal interface (Cobra + Charm)                            | —    |
| bookie-breaker-ui                 | TypeScript/SvelteKit | Web dashboard                                                 | 3000 |
| bookie-breaker-infra-ops          | Docker/YAML          | Docker Compose, CI/CD, scripts                                | —    |
| bookie-breaker-docs               | Markdown             | Architecture, schemas, operations docs                        | —    |

## Communication Patterns

- **Synchronous:** REST APIs between services (`/api/v1/{resource}`)
- **Asynchronous:** Redis pub/sub for events (fire-and-forget, no persistence)
- **Caching:** Redis with TTLs for stats, simulation results, dashboard data

### Event Channels

- `events:lines.updated` — Line changes detected
- `events:stats.updated` — Injury or stat updates
- `events:game.completed` — Game final scores
- `events:simulation.completed` — Simulation finished
- `events:prediction.completed` — Predictions generated
- `events:edge.detected` — Edge identified

## Data Flow

```
statistics-service → simulation-engine → prediction-engine → agent → bookie-emulator
                                                                ↑
lines-service ─────────────────────────────────────────────────┘
```

1. statistics-service fetches team/player stats from external APIs
2. simulation-engine runs Monte Carlo simulations using those stats
3. prediction-engine calibrates probabilities with ML, adding contextual features
4. agent compares calibrated probabilities to market lines from lines-service
5. agent detects edges and places paper bets via bookie-emulator

## Data Ownership

| Entity                               | Owner              | Others Read Via |
| ------------------------------------ | ------------------ | --------------- |
| Teams, Players, Games, Venues        | statistics-service | REST API        |
| Sportsbooks, BettingLines            | lines-service      | REST API        |
| SimulationConfigs, SimulationResults | simulation-engine  | REST API        |
| Predictions, Models, Features        | prediction-engine  | REST API        |
| Edges, Alerts, Analyses              | agent              | REST API        |
| PaperBets, BetGrades, Bankroll       | bookie-emulator    | REST API        |

## Database Architecture

- **PostgreSQL 16 + TimescaleDB** with schema-per-service: `lines`, `predictions`, `emulator`
- **Redis 7** for caching and pub/sub events
- Cross-service references use UUIDs only, never direct DB access

## Repository Layout

All 11 repos live under `BookieBreaker/`:

- Documentation: `bookie-breaker-docs/` (architecture, schemas, ADRs, operations)
- Infrastructure: `bookie-breaker-infra-ops/` (Docker Compose, CI/CD)
- Services: `bookie-breaker-{service}/` (each self-contained)

## Conventions

### Commits

Conventional Commits enforced by commitlint: `type(scope): description`
Types: feat, fix, chore, docs, refactor, test, perf, ci, build

### API Design

- All endpoints: `/api/v1/{resource}`
- Standard JSON envelope: `{ "data": ..., "meta": { "timestamp", "request_id" } }`
- UUID v4 for all identifiers
- Pagination via `?limit=&offset=`

### Tooling

- **mise** for tool version pinning (`.config/mise.toml`)
- **lefthook** for git hooks (`.config/lefthook.yml`)
- **go-task** for task automation (`Taskfile.yml`)
- **Renovate** for dependency updates (`renovate.json`)

### Testing

- `tests/unit/` — No external deps, fast
- `tests/integration/` — Real Postgres/Redis (testcontainers)
- `tests/e2e/` — Full service stack

### Environment Variables

- Set `MISE_CONFIG_FILE=.config/mise.toml` in your shell profile
- Set `LEFTHOOK_CONFIG=.config/lefthook.yml` in your shell profile
- Copy `.env.example` to `.env` per repo for local dev

## Shared Commands

From `BookieBreaker/` root:

- `task bootstrap` — Install tools and hooks across all repos
- `task up` — Start full stack via Docker Compose
- `task down` — Stop all services
- `task build` — Build all containers
- `task test` — Run tests across all repos
- `task lint` — Lint all repos
- `task logs` — Aggregate logs

Per-repo:

- `task bootstrap` — Install tools and hooks for this repo
- `task dev` — Run with hot reload
- `task lint` — Run linters
- `task test` — Run tests
