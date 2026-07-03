# PLANNING: lines-service

## Service

- **Name:** lines-service
- **Language:** Go 1.22+
- **Framework:** Echo

## Implementation Phase

Phase 1 (Infrastructure & Data Foundation)

## Purpose

Ingests betting lines and odds from The Odds API (and optionally SharpAPI SSE), normalizes odds formats, stores line
snapshots with timestamps in TimescaleDB, and serves current/historical lines to all other services and interfaces.

## Ordered Task List

- [x] Initialize Go module, set up project structure: `cmd/server/`, `internal/`, `pkg/`
- [x] Set up Echo HTTP server with middleware (logging, recovery, CORS, request ID)
- [x] Implement health check endpoint (`GET /healthz`)
- [x] Implement Postgres (pgx) connection with TimescaleDB support
- [x] Design database schema: `sportsbooks` table, `line_snapshots` hypertable (TimescaleDB), `closing_lines` table
      (no `games` table — snapshots carry `game_external_id`; game entities are owned by statistics-service)
- [x] Implement database migrations (golang-migrate, per [ADR-019](../../decisions/019-database-migration-tooling.md))
- [x] Implement The Odds API client: fetch current odds for a sport, respect rate limits (quota warning at <50
      requests remaining)
- [x] Implement line normalization: convert American/decimal/fractional odds to canonical format, normalize market types
      (spread, total, moneyline)
- [x] Implement ingestion scheduler: configurable poll interval (default 5 minutes); polls sports sequentially
      (concurrency deferred)
- [ ] Implement deduplication: detect and skip unchanged lines to avoid redundant snapshots
- [x] Set up TimescaleDB hypertable for `line_snapshots` with compression and retention policies
- [ ] Implement closing line detection: capture final line before game start time (table and repository exist;
      detection logic not implemented)
- [ ] Build REST API endpoints (per the OpenAPI spec):
  - [ ] `GET /api/v1/lines/current` -- current lines with filters (league, game, market type, sportsbook)
  - [ ] `GET /api/v1/lines/game/{game_id}` -- all lines for a game
  - [ ] `GET /api/v1/lines/game/{game_id}/movement` -- line movement history
  - [ ] `GET /api/v1/lines/game/{game_id}/best` -- best available line per market
  - [ ] `GET /api/v1/lines/game/{game_id}/closing` -- closing lines for a completed game
  - [ ] `GET /api/v1/lines/snapshots/{line_id}` -- single line snapshot
  - [ ] `GET /api/v1/lines/sportsbooks` -- sportsbook registry
  - [ ] `POST /api/v1/lines/ingestion/trigger` -- manual ingestion trigger
- [x] Implement Redis caching for current lines (cache layer with per-game invalidation on ingest; read-path usage
      lands with the REST endpoints)
- [x] Implement Redis pub/sub: publish `lines.updated` events on new line snapshots
- [x] Add OpenAPI spec (hand-authored in bookie-breaker-docs `api-contracts/openapi/lines-service.yaml`, spec-first
      per [ADR-021](../../decisions/021-openapi-spec-strategy.md))
- [ ] Write unit tests for odds normalization and deduplication logic (normalizer covered; dedup logic not built yet)
- [ ] Write integration tests against real Postgres+TimescaleDB and Redis
- [x] Create Dockerfile (multi-stage build) and `.config/air.toml` for hot reload
- [x] Add `.env.example`

**Phase 7 additions (advanced):**

- [ ] Integrate SharpAPI SSE stream for real-time in-game line updates
- [ ] Add player prop and team prop line ingestion
- [ ] Implement sharp move and steam move detection

## Dependencies

- **infra-ops** must provide Docker Compose with Postgres+TimescaleDB and Redis running
- **The Odds API key** required (environment variable)
- No upstream service dependencies

## Complexity

**L** -- Focused scope: one primary external API, well-defined storage model, straightforward REST API. TimescaleDB adds
some complexity for hypertable management and compression policies.

## Definition of Done

- [ ] `GET /api/v1/lines/current?league=nba` returns live NBA odds from multiple sportsbooks
- [ ] Line snapshots are persisted in TimescaleDB with timestamps
- [ ] `GET /api/v1/lines/game/{game_id}/movement` returns chronological line movement
- [ ] Closing lines are captured for completed games
- [x] Odds are normalized to a canonical format (American odds + implied probability)
- [x] Ingestion scheduler runs at configurable intervals
- [ ] Deduplication prevents redundant snapshots
- [x] `lines.updated` events are published to Redis pub/sub
- [x] Health check endpoint responds
- [x] OpenAPI spec is valid
- [x] Tests pass

## Key Documentation

- [Lines Service Component](../bookie-breaker-docs/components/lines-service.md)
- [Lines Data Sources (ADR-007)](../bookie-breaker-docs/decisions/007-lines-data-sources.md)
- [Lines Data Sources Research](../bookie-breaker-docs/research/lines-data-sources.md)
- [Feature Inventory: PIPE-008 through PIPE-014](../bookie-breaker-docs/architecture/feature-inventory.md)
- [Database Schemas](../bookie-breaker-docs/schemas/database-schemas/)
