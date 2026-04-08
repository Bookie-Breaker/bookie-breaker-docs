# Development Workflow

How to set up, run, and develop BookieBreaker locally as a solo developer managing 11 repositories.

---

## 1. Repository Organization

### Directory Layout

All 11 repos live under a single `BookieBreaker/` root directory. The docs repo and infra-ops repo are peers alongside the service repos:

```
BookieBreaker/
├── bookie-breaker-docs/          # Architecture, ADRs, schemas, operations docs
├── bookie-breaker-infra-ops/     # Docker Compose, Taskfile, CI/CD, Kustomize
├── bookie-breaker-lines-service/         # Go — line ingestion and serving
├── bookie-breaker-statistics-service/    # Go — stats ingestion, caching, enrichment
├── bookie-breaker-simulation-engine/     # Python — Monte Carlo simulations
├── bookie-breaker-prediction-engine/     # Python — ML calibration, edge detection
├── bookie-breaker-agent/                 # Python — LLM orchestration, pipeline coordinator
├── bookie-breaker-mcp-server/            # Python — MCP tool server
├── bookie-breaker-bookie-emulator/       # Python — paper trading, performance tracking
├── bookie-breaker-cli/                   # Go — terminal interface (Charm ecosystem)
├── bookie-breaker-ui/                    # TypeScript/SvelteKit — web dashboard
└── Taskfile.yml                          # Root Taskfile for cross-repo operations
```

### Clone Script

Create `BookieBreaker/clone-all.sh` to bootstrap the workspace:

```bash
#!/usr/bin/env bash
set -euo pipefail

GITHUB_ORG="your-github-username"  # Update this
BASE_DIR="$(cd "$(dirname "$0")" && pwd)"

REPOS=(
  bookie-breaker-docs
  bookie-breaker-infra-ops
  bookie-breaker-lines-service
  bookie-breaker-statistics-service
  bookie-breaker-simulation-engine
  bookie-breaker-prediction-engine
  bookie-breaker-agent
  bookie-breaker-mcp-server
  bookie-breaker-bookie-emulator
  bookie-breaker-cli
  bookie-breaker-ui
)

for repo in "${REPOS[@]}"; do
  if [ -d "$BASE_DIR/$repo" ]; then
    echo "Skipping $repo (already exists)"
  else
    echo "Cloning $repo..."
    git clone "git@github.com:${GITHUB_ORG}/${repo}.git" "$BASE_DIR/$repo"
  fi
done

echo "All repos cloned. Run 'task up' from $BASE_DIR to start the stack."
```

---

## 2. Docker Compose Setup

The Docker Compose file lives in `bookie-breaker-infra-ops/docker-compose.yml` and defines the full stack. The root `Taskfile.yml` delegates to it.

### Port Assignments

| Service            | Port | Protocol  |
| ------------------ | ---- | --------- |
| lines-service      | 8001 | HTTP/REST |
| statistics-service | 8002 | HTTP/REST |
| simulation-engine  | 8003 | HTTP/REST |
| prediction-engine  | 8004 | HTTP/REST |
| bookie-emulator    | 8005 | HTTP/REST |
| agent              | 8006 | HTTP/REST |
| mcp-server         | 8007 | HTTP/REST |
| ui                 | 3000 | HTTP      |
| PostgreSQL         | 5432 | TCP       |
| Redis              | 6379 | TCP       |

### Docker Compose Structure

```yaml
# bookie-breaker-infra-ops/docker-compose.yml
version: "3.9"

networks:
  bookiebreaker:
    driver: bridge

volumes:
  postgres-data:
  redis-data:

services:
  # ── Databases ───────────────────────────────────────
  postgres:
    image: timescale/timescaledb:latest-pg16
    container_name: bb-postgres
    environment:
      POSTGRES_USER: bookiebreaker
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-localdev}
      POSTGRES_DB: bookiebreaker
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-db:/docker-entrypoint-initdb.d # Schema creation scripts
    networks:
      - bookiebreaker
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U bookiebreaker"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: bb-redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - bookiebreaker
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  # ── Go Services ─────────────────────────────────────
  lines-service:
    build:
      context: ../bookie-breaker-lines-service
      dockerfile: Dockerfile
    container_name: bb-lines-service
    ports:
      - "8001:8001"
    volumes:
      - ../bookie-breaker-lines-service:/app # Hot reload with air
    environment:
      - PORT=8001
      - DATABASE_URL=postgres://bookiebreaker:${POSTGRES_PASSWORD:-localdev}@postgres:5432/bookiebreaker?search_path=lines
      - REDIS_URL=redis://redis:6379
      - ODDS_API_KEY=${ODDS_API_KEY}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - bookiebreaker

  statistics-service:
    build:
      context: ../bookie-breaker-statistics-service
      dockerfile: Dockerfile
    container_name: bb-statistics-service
    ports:
      - "8002:8002"
    volumes:
      - ../bookie-breaker-statistics-service:/app
    environment:
      - PORT=8002
      - REDIS_URL=redis://redis:6379
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - bookiebreaker

  # ── Python Services ─────────────────────────────────
  simulation-engine:
    build:
      context: ../bookie-breaker-simulation-engine
      dockerfile: Dockerfile
    container_name: bb-simulation-engine
    ports:
      - "8003:8003"
    volumes:
      - ../bookie-breaker-simulation-engine/src:/app/src # Hot reload with uvicorn
    environment:
      - PORT=8003
      - REDIS_URL=redis://redis:6379
      - STATISTICS_SERVICE_URL=http://statistics-service:8002
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - bookiebreaker

  prediction-engine:
    build:
      context: ../bookie-breaker-prediction-engine
      dockerfile: Dockerfile
    container_name: bb-prediction-engine
    ports:
      - "8004:8004"
    volumes:
      - ../bookie-breaker-prediction-engine/src:/app/src
    environment:
      - PORT=8004
      - DATABASE_URL=postgres://bookiebreaker:${POSTGRES_PASSWORD:-localdev}@postgres:5432/bookiebreaker?search_path=predictions
      - REDIS_URL=redis://redis:6379
      - STATISTICS_SERVICE_URL=http://statistics-service:8002
      - LINES_SERVICE_URL=http://lines-service:8001
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - bookiebreaker

  bookie-emulator:
    build:
      context: ../bookie-breaker-bookie-emulator
      dockerfile: Dockerfile
    container_name: bb-bookie-emulator
    ports:
      - "8005:8005"
    volumes:
      - ../bookie-breaker-bookie-emulator/src:/app/src
    environment:
      - PORT=8005
      - DATABASE_URL=postgres://bookiebreaker:${POSTGRES_PASSWORD:-localdev}@postgres:5432/bookiebreaker?search_path=emulator
      - REDIS_URL=redis://redis:6379
      - LINES_SERVICE_URL=http://lines-service:8001
      - STATISTICS_SERVICE_URL=http://statistics-service:8002
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - bookiebreaker

  agent:
    build:
      context: ../bookie-breaker-agent
      dockerfile: Dockerfile
    container_name: bb-agent
    ports:
      - "8006:8006"
    volumes:
      - ../bookie-breaker-agent/src:/app/src
    environment:
      - PORT=8006
      - REDIS_URL=redis://redis:6379
      - LINES_SERVICE_URL=http://lines-service:8001
      - STATISTICS_SERVICE_URL=http://statistics-service:8002
      - SIMULATION_ENGINE_URL=http://simulation-engine:8003
      - PREDICTION_ENGINE_URL=http://prediction-engine:8004
      - BOOKIE_EMULATOR_URL=http://bookie-emulator:8005
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - bookiebreaker

  mcp-server:
    build:
      context: ../bookie-breaker-mcp-server
      dockerfile: Dockerfile
    container_name: bb-mcp-server
    ports:
      - "8007:8007"
    volumes:
      - ../bookie-breaker-mcp-server/src:/app/src
    environment:
      - PORT=8007
      - AGENT_URL=http://agent:8006
      - LINES_SERVICE_URL=http://lines-service:8001
      - STATISTICS_SERVICE_URL=http://statistics-service:8002
      - BOOKIE_EMULATOR_URL=http://bookie-emulator:8005
    depends_on:
      - agent
    networks:
      - bookiebreaker

  # ── TypeScript/SvelteKit UI ─────────────────────────
  ui:
    build:
      context: ../bookie-breaker-ui
      dockerfile: Dockerfile
    container_name: bb-ui
    ports:
      - "3000:3000"
    volumes:
      - ../bookie-breaker-ui/src:/app/src # Vite hot reload
      - ../bookie-breaker-ui/static:/app/static
    environment:
      - PUBLIC_AGENT_URL=http://agent:8006
      - PUBLIC_LINES_SERVICE_URL=http://lines-service:8001
      - PUBLIC_STATISTICS_SERVICE_URL=http://statistics-service:8002
      - PUBLIC_BOOKIE_EMULATOR_URL=http://bookie-emulator:8005
      - PUBLIC_REDIS_URL=redis://redis:6379
    depends_on:
      - agent
    networks:
      - bookiebreaker
```

### Database Init Scripts

Place schema creation scripts in `bookie-breaker-infra-ops/init-db/`:

```
init-db/
├── 01-create-schemas.sql     # CREATE SCHEMA lines, predictions, emulator;
├── 02-create-roles.sql       # Per-service Postgres roles with schema-scoped access
└── 03-create-shared-enums.sql # league_enum, market_type_enum, etc.
```

---

## 3. Taskfile Commands

The root `Taskfile.yml` sits at the `BookieBreaker/` level and orchestrates all 11 repos:

```yaml
# BookieBreaker/Taskfile.yml
version: "3"

vars:
  COMPOSE_FILE: ./bookie-breaker-infra-ops/docker-compose.yml
  COMPOSE_CMD: docker compose -f {{.COMPOSE_FILE}}

tasks:
  up:
    desc: Start all services via Docker Compose
    cmds:
      - "{{.COMPOSE_CMD}} up -d"

  down:
    desc: Stop all services and remove containers
    cmds:
      - "{{.COMPOSE_CMD}} down"

  build:
    desc: Build (or rebuild) all containers
    cmds:
      - "{{.COMPOSE_CMD}} build"

  test:
    desc: Run tests across all repos
    cmds:
      - task: test:go
      - task: test:python
      - task: test:ts

  test:go:
    desc: Run Go tests
    cmds:
      - cd bookie-breaker-lines-service && go test ./...
      - cd bookie-breaker-statistics-service && go test ./...
      - cd bookie-breaker-cli && go test ./...

  test:python:
    desc: Run Python tests
    cmds:
      - cd bookie-breaker-simulation-engine && uv run pytest --cov
      - cd bookie-breaker-prediction-engine && uv run pytest --cov
      - cd bookie-breaker-agent && uv run pytest --cov
      - cd bookie-breaker-mcp-server && uv run pytest --cov
      - cd bookie-breaker-bookie-emulator && uv run pytest --cov

  test:ts:
    desc: Run TypeScript tests
    cmds:
      - cd bookie-breaker-ui && pnpm test

  lint:
    desc: Lint all repos
    cmds:
      - task: lint:go
      - task: lint:python
      - task: lint:ts

  lint:go:
    desc: Lint Go repos with golangci-lint
    cmds:
      - cd bookie-breaker-lines-service && golangci-lint run
      - cd bookie-breaker-statistics-service && golangci-lint run
      - cd bookie-breaker-cli && golangci-lint run

  lint:python:
    desc: Lint Python repos with ruff
    cmds:
      - cd bookie-breaker-simulation-engine && uv run ruff check . && uv run ruff format --check .
      - cd bookie-breaker-prediction-engine && uv run ruff check . && uv run ruff format --check .
      - cd bookie-breaker-agent && uv run ruff check . && uv run ruff format --check .
      - cd bookie-breaker-mcp-server && uv run ruff check . && uv run ruff format --check .
      - cd bookie-breaker-bookie-emulator && uv run ruff check . && uv run ruff format --check .

  lint:ts:
    desc: Lint TypeScript/SvelteKit with ESLint + Prettier
    cmds:
      - cd bookie-breaker-ui && pnpm lint && pnpm format --check

  gen:
    desc: Regenerate OpenAPI clients from specs
    cmds:
      - echo "Regenerating Go clients from Python service OpenAPI specs..."
      - cd bookie-breaker-lines-service && go generate ./...
      - cd bookie-breaker-statistics-service && go generate ./...
      - cd bookie-breaker-cli && go generate ./...
      - echo "Regenerating TypeScript clients..."
      - cd bookie-breaker-ui && pnpm gen:api

  logs:
    desc: Tail logs from all services
    cmds:
      - "{{.COMPOSE_CMD}} logs -f"

  logs:service:
    desc: "Tail logs from a specific service (usage: task logs:service -- lines-service)"
    cmds:
      - "{{.COMPOSE_CMD}} logs -f {{.CLI_ARGS}}"

  db:migrate:
    desc: Run database migrations for all Postgres-backed services
    cmds:
      - cd bookie-breaker-lines-service && go run cmd/migrate/main.go up
      - cd bookie-breaker-prediction-engine && uv run alembic upgrade head
      - cd bookie-breaker-bookie-emulator && uv run alembic upgrade head

  db:seed:
    desc: Seed databases with development fixture data
    cmds:
      - cd bookie-breaker-infra-ops && ./scripts/seed-data.sh

  db:reset:
    desc: Drop and recreate all schemas, run migrations, then seed
    cmds:
      - task: down
      - docker volume rm -f bookie-breaker-infra-ops_postgres-data
      - task: up
      - sleep 5
      - task: db:migrate
      - task: db:seed

  # ── Per-service dev mode ────────────────────────────
  lines-service:dev:
    desc: Run lines-service locally with air (hot reload)
    dir: bookie-breaker-lines-service
    cmds:
      - air

  statistics-service:dev:
    desc: Run statistics-service locally with air (hot reload)
    dir: bookie-breaker-statistics-service
    cmds:
      - air

  simulation-engine:dev:
    desc: Run simulation-engine locally with uvicorn (hot reload)
    dir: bookie-breaker-simulation-engine
    cmds:
      - uv run uvicorn src.main:app --reload --port 8003

  prediction-engine:dev:
    desc: Run prediction-engine locally with uvicorn (hot reload)
    dir: bookie-breaker-prediction-engine
    cmds:
      - uv run uvicorn src.main:app --reload --port 8004

  bookie-emulator:dev:
    desc: Run bookie-emulator locally with uvicorn (hot reload)
    dir: bookie-breaker-bookie-emulator
    cmds:
      - uv run uvicorn src.main:app --reload --port 8005

  agent:dev:
    desc: Run agent locally with uvicorn (hot reload)
    dir: bookie-breaker-agent
    cmds:
      - uv run uvicorn src.main:app --reload --port 8006

  mcp-server:dev:
    desc: Run mcp-server locally with uvicorn (hot reload)
    dir: bookie-breaker-mcp-server
    cmds:
      - uv run uvicorn src.main:app --reload --port 8007

  cli:dev:
    desc: Build and run CLI in watch mode
    dir: bookie-breaker-cli
    cmds:
      - go run ./cmd/cli

  ui:dev:
    desc: Run SvelteKit UI in dev mode (Vite hot reload)
    dir: bookie-breaker-ui
    cmds:
      - pnpm dev
```

---

## 4. Hot Reload per Language

### Go (lines-service, statistics-service, cli)

Use [air](https://github.com/air-verse/air) for automatic rebuild on file change. Each Go service repo contains a `.air.toml`:

```toml
# .air.toml (in each Go service repo root)
root = "."
tmp_dir = "tmp"

[build]
  cmd = "go build -o ./tmp/main ./cmd/server"
  bin = "./tmp/main"
  include_ext = ["go", "tpl", "tmpl", "html"]
  exclude_dir = ["tmp", "vendor", "testdata"]
  delay = 1000  # ms

[log]
  time = false
```

In Docker, the Dockerfile dev target installs air and runs it instead of the compiled binary. The volume mount (`../bookie-breaker-lines-service:/app`) ensures file changes on the host trigger rebuilds inside the container.

### Python (simulation-engine, prediction-engine, agent, mcp-server, bookie-emulator)

Use `uvicorn --reload` for automatic restart on file change. Each Python service Dockerfile dev target runs:

```dockerfile
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8003", "--reload"]
```

The volume mount maps `src/` into the container so edits are picked up immediately. Uvicorn watches for `.py` file changes by default.

### TypeScript/SvelteKit (ui)

SvelteKit uses Vite's built-in HMR (Hot Module Replacement):

```dockerfile
CMD ["pnpm", "dev", "--host", "0.0.0.0", "--port", "3000"]
```

The volume mount maps `src/` and `static/` so component changes reflect instantly in the browser without a full page reload.

---

## 5. Standalone Development

When focusing on a single service, run it outside Docker while pointing at shared infrastructure (Postgres + Redis) running in Docker.

### Start only infrastructure

```bash
# From BookieBreaker/
docker compose -f bookie-breaker-infra-ops/docker-compose.yml up -d postgres redis
```

### Per-service standalone setup

| Service            | Command                                                                                   | Dependencies to mock/stub                                                                             |
| ------------------ | ----------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| lines-service      | `cd bookie-breaker-lines-service && air`                                                  | Set `ODDS_API_KEY` in `.env`. Postgres + Redis must be running. No service dependencies.              |
| statistics-service | `cd bookie-breaker-statistics-service && air`                                             | Redis must be running. External stats APIs are called directly (internet required).                   |
| simulation-engine  | `cd bookie-breaker-simulation-engine && uv run uvicorn src.main:app --reload --port 8003` | Needs statistics-service running (or stub responses via `STATISTICS_SERVICE_URL`). Redis for caching. |
| prediction-engine  | `cd bookie-breaker-prediction-engine && uv run uvicorn src.main:app --reload --port 8004` | Needs statistics-service + lines-service (or stubs). Postgres + Redis.                                |
| agent              | `cd bookie-breaker-agent && uv run uvicorn src.main:app --reload --port 8006`             | Needs all backend services. Set `ANTHROPIC_API_KEY`. Best run against the full Docker stack.          |
| bookie-emulator    | `cd bookie-breaker-bookie-emulator && uv run uvicorn src.main:app --reload --port 8005`   | Postgres + Redis. Needs lines-service + statistics-service for grading.                               |
| mcp-server         | `cd bookie-breaker-mcp-server && uv run uvicorn src.main:app --reload --port 8007`        | Needs agent running.                                                                                  |
| cli                | `cd bookie-breaker-cli && go run ./cmd/cli`                                               | Needs agent running. Direct lookups need lines-service, statistics-service, bookie-emulator.          |
| ui                 | `cd bookie-breaker-ui && pnpm dev`                                                        | Needs agent running. Direct lookups need data services.                                               |

### Stubbing dependencies

For isolated development without running upstream services:

1. **HTTP stubbing:** Use a lightweight mock server (e.g., [mountebank](http://www.mbtest.org/) or [WireMock](https://wiremock.org/)) that returns canned JSON responses matching the OpenAPI contracts.
2. **Environment override:** Point `*_SERVICE_URL` env vars at the mock server instead of real services.
3. **Recorded responses:** Record real responses once, replay them locally. See the testing strategy doc for the VCR pattern.

---

## 6. Seed Data Strategy

### What gets seeded

The seed script populates Postgres and Redis with enough data to exercise all APIs end-to-end:

| Data               | Quantity                                 | Purpose                               |
| ------------------ | ---------------------------------------- | ------------------------------------- |
| Sportsbooks        | 5-10 (FanDuel, DraftKings, BetMGM, etc.) | Lines need sportsbook references      |
| Games              | 20-30 per league, 2 leagues (NFL, NBA)   | Enough to show lists, filters, trends |
| Line snapshots     | 5-10 snapshots per game per sportsbook   | Shows line movement over time         |
| Closing lines      | 1 per completed game/market/sportsbook   | Historical comparison for CLV         |
| Team stats (Redis) | Current season stats for seeded teams    | Simulation and prediction inputs      |
| Predictions        | 50-100 with varied confidence levels     | Dashboard and performance views       |
| Model versions     | 2-3 versions                             | Model comparison UI                   |
| Paper bets         | 30-50 (mix of OPEN, WON, LOST, PUSH)     | Bookie emulator performance charts    |
| Bankroll snapshots | 30 daily snapshots                       | Bankroll over time chart              |

### Seed script

```bash
# bookie-breaker-infra-ops/scripts/seed-data.sh
#!/usr/bin/env bash
set -euo pipefail

DB_URL="${DATABASE_URL:-postgres://bookiebreaker:localdev@localhost:5432/bookiebreaker}"

echo "Seeding lines schema..."
psql "$DB_URL" -f ./fixtures/lines-seed.sql

echo "Seeding predictions schema..."
psql "$DB_URL" -f ./fixtures/predictions-seed.sql

echo "Seeding emulator schema..."
psql "$DB_URL" -f ./fixtures/emulator-seed.sql

echo "Seeding Redis cache..."
python3 ./fixtures/redis-seed.py

echo "Seed complete."
```

### Fixture files

Store SQL fixture files in `bookie-breaker-infra-ops/fixtures/`:

```
fixtures/
├── lines-seed.sql          # Sportsbooks, games, line_snapshots, closing_lines
├── predictions-seed.sql    # Model versions, predictions, feature vectors
├── emulator-seed.sql       # Paper bets, bankroll snapshots
└── redis-seed.py           # Populate Redis with cached team stats, simulation results
```

Use realistic but synthetic data. Real team names and plausible stat lines make the dashboard look meaningful during development. Timestamps should cover a two-week window around the current date so time-based queries return results.

---

## 7. Environment Variables

### `.env.example` pattern

Each repo contains a `.env.example` with every variable the service reads, documented with inline comments. Copy to `.env` and fill in real values. The `.env` file is `.gitignore`d.

The infra-ops repo contains a root `.env.example` used by Docker Compose:

```bash
# bookie-breaker-infra-ops/.env.example

# ── Database ──────────────────────────────────────────
POSTGRES_PASSWORD=localdev

# ── External API Keys ────────────────────────────────
ODDS_API_KEY=your-odds-api-key-here
ANTHROPIC_API_KEY=your-anthropic-api-key-here

# ── Optional overrides ───────────────────────────────
# LOG_LEVEL=debug
# SIMULATION_ITERATIONS=1000     # Lower for faster dev runs (prod: 10000-50000)
# REDIS_URL=redis://redis:6379   # Override if running Redis elsewhere
```

### Per-service environment variables

#### All services (common)

| Variable    | Description             | Default                       |
| ----------- | ----------------------- | ----------------------------- |
| `PORT`      | HTTP listen port        | Per service (8001-8007, 3000) |
| `LOG_LEVEL` | Logging verbosity       | `info`                        |
| `REDIS_URL` | Redis connection string | `redis://localhost:6379`      |

#### Services with Postgres (lines-service, prediction-engine, bookie-emulator)

| Variable       | Description                                             | Default         |
| -------------- | ------------------------------------------------------- | --------------- |
| `DATABASE_URL` | Full Postgres connection string including `search_path` | None (required) |

#### lines-service

| Variable                 | Description              | Default         |
| ------------------------ | ------------------------ | --------------- |
| `ODDS_API_KEY`           | API key for The Odds API | None (required) |
| `ODDS_API_POLL_INTERVAL` | Seconds between polls    | `300`           |
| `SHARP_API_URL`          | SharpAPI SSE endpoint    | None (optional) |

#### agent

| Variable                 | Description                     | Default                          |
| ------------------------ | ------------------------------- | -------------------------------- |
| `ANTHROPIC_API_KEY`      | Anthropic API key for LLM calls | None (required)                  |
| `LINES_SERVICE_URL`      | URL of lines-service            | `http://lines-service:8001`      |
| `STATISTICS_SERVICE_URL` | URL of statistics-service       | `http://statistics-service:8002` |
| `SIMULATION_ENGINE_URL`  | URL of simulation-engine        | `http://simulation-engine:8003`  |
| `PREDICTION_ENGINE_URL`  | URL of prediction-engine        | `http://prediction-engine:8004`  |
| `BOOKIE_EMULATOR_URL`    | URL of bookie-emulator          | `http://bookie-emulator:8005`    |

#### simulation-engine

| Variable                 | Description                        | Default                          |
| ------------------------ | ---------------------------------- | -------------------------------- |
| `STATISTICS_SERVICE_URL` | URL of statistics-service          | `http://statistics-service:8002` |
| `SIMULATION_ITERATIONS`  | Monte Carlo iterations per matchup | `10000`                          |

#### prediction-engine

| Variable                 | Description                 | Default                          |
| ------------------------ | --------------------------- | -------------------------------- |
| `STATISTICS_SERVICE_URL` | URL of statistics-service   | `http://statistics-service:8002` |
| `LINES_SERVICE_URL`      | URL of lines-service        | `http://lines-service:8001`      |
| `MODEL_DIR`              | Path to trained model files | `./models`                       |

#### mcp-server

| Variable                 | Description                 | Default                          |
| ------------------------ | --------------------------- | -------------------------------- |
| `AGENT_URL`              | URL of the agent service    | `http://agent:8006`              |
| `LINES_SERVICE_URL`      | URL for direct data lookups | `http://lines-service:8001`      |
| `STATISTICS_SERVICE_URL` | URL for direct data lookups | `http://statistics-service:8002` |
| `BOOKIE_EMULATOR_URL`    | URL for direct data lookups | `http://bookie-emulator:8005`    |

#### ui (SvelteKit public env vars)

| Variable                        | Description                           | Default                 |
| ------------------------------- | ------------------------------------- | ----------------------- |
| `PUBLIC_AGENT_URL`              | Agent API URL (browser-accessible)    | `http://localhost:8006` |
| `PUBLIC_LINES_SERVICE_URL`      | Lines API URL (browser-accessible)    | `http://localhost:8001` |
| `PUBLIC_STATISTICS_SERVICE_URL` | Stats API URL (browser-accessible)    | `http://localhost:8002` |
| `PUBLIC_BOOKIE_EMULATOR_URL`    | Emulator API URL (browser-accessible) | `http://localhost:8005` |

### Local vs Docker vs Production

| Concern               | Local (standalone)        | Docker Compose                              | Production                        |
| --------------------- | ------------------------- | ------------------------------------------- | --------------------------------- |
| Service URLs          | `http://localhost:{port}` | `http://{service-name}:{port}` (Docker DNS) | Kubernetes service DNS or ingress |
| Database              | `localhost:5432`          | `postgres:5432` (container name)            | Managed Postgres endpoint         |
| Redis                 | `localhost:6379`          | `redis:6379` (container name)               | Managed Redis endpoint            |
| API keys              | `.env` file per repo      | Root `.env` loaded by Compose               | Kubernetes secrets / Vault        |
| Log level             | `debug`                   | `info`                                      | `warn`                            |
| Simulation iterations | `1000` (fast feedback)    | `5000` (balanced)                           | `10000-50000` (full accuracy)     |
