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

- [ ] Initialize Go module, set up project structure: `cmd/server/`, `internal/`, `pkg/`
- [ ] Set up Echo HTTP server with middleware (logging, recovery, CORS, request ID)
- [ ] Implement health check endpoint (`GET /healthz`)
- [ ] Implement Postgres (pgx) connection with TimescaleDB support
- [ ] Design database schema: `sportsbooks` table, `games` table, `line_snapshots` hypertable (TimescaleDB),
      `closing_lines` table
- [ ] Implement database migrations (golang-migrate, per [ADR-019](../../decisions/019-database-migration-tooling.md))
- [ ] Implement The Odds API client: fetch current odds for a sport, handle pagination, respect rate limits
- [ ] Implement line normalization: convert American/decimal/fractional odds to canonical format, normalize market types
      (spread, total, moneyline)
- [ ] Implement ingestion scheduler: configurable poll interval (default 5 minutes), concurrent polling for multiple
      sports
- [ ] Implement deduplication: detect and skip unchanged lines to avoid redundant snapshots
- [ ] Set up TimescaleDB hypertable for `line_snapshots` with compression and retention policies
- [ ] Implement closing line detection: capture final line before game start time
- [ ] Build REST API endpoints:
  - [ ] `GET /api/v1/lines/current` -- current lines with filters (sport, game, market type, sportsbook)
  - [ ] `GET /api/v1/lines/history` -- line movement history for a specific game/market
  - [ ] `GET /api/v1/lines/closing` -- closing lines for completed games
  - [ ] `GET /api/v1/lines/games` -- list of games with available lines
- [ ] Implement Redis caching for current lines (short TTL, refreshed on each poll)
- [ ] Implement Redis pub/sub: publish `lines.updated` events on new line snapshots
- [ ] Add OpenAPI spec generation
- [ ] Write unit tests for odds normalization and deduplication logic
- [ ] Write integration tests against real Postgres+TimescaleDB and Redis
- [ ] Create Dockerfile (multi-stage build) and `.air.toml` for hot reload
- [ ] Add `.env.example`

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

- [ ] `GET /api/v1/lines/current?sport=basketball_nba` returns live NBA odds from multiple sportsbooks
- [ ] Line snapshots are persisted in TimescaleDB with timestamps
- [ ] `GET /api/v1/lines/history?game_id=X` returns chronological line movement
- [ ] Closing lines are captured for completed games
- [ ] Odds are normalized to a canonical format (American odds + implied probability)
- [ ] Ingestion scheduler runs at configurable intervals
- [ ] Deduplication prevents redundant snapshots
- [ ] `lines.updated` events are published to Redis pub/sub
- [ ] Health check endpoint responds
- [ ] OpenAPI spec is valid
- [ ] Tests pass

## Key Documentation

- [Lines Service Component](../bookie-breaker-docs/components/lines-service.md)
- [Lines Data Sources (ADR-007)](../bookie-breaker-docs/decisions/007-lines-data-sources.md)
- [Lines Data Sources Research](../bookie-breaker-docs/research/lines-data-sources.md)
- [Feature Inventory: PIPE-008 through PIPE-014](../bookie-breaker-docs/architecture/feature-inventory.md)
- [Database Schemas](../bookie-breaker-docs/schemas/database-schemas/)
