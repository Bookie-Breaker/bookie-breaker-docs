# PLANNING: statistics-service

## Service

- **Name:** statistics-service
- **Language:** Go 1.22+
- **Framework:** Echo

## Implementation Phase

Phase 1 (Infrastructure & Data Foundation)

## Purpose

Ingests, normalizes, caches, and serves sports statistics from external APIs. Acts as a cache + enrichment layer over
external data sources (nba_api, nfl_data_py, pybaseball, CFBD). Computes derived statistics like rolling averages,
efficiency ratings, and advanced metrics.

## Ordered Task List

- [x] Initialize Go module, set up project structure: `cmd/server/`, `internal/`, `pkg/`
- [x] Set up Echo HTTP server with middleware (logging, recovery, CORS, request ID)
- [x] Implement health check endpoint (`GET /api/v1/stats/health` per the OpenAPI spec; Redis drives unhealthy,
      Postgres degrades, upstream freshness is reported from last-success timestamps without external calls)
- [x] Implement Redis client connection and caching layer with configurable TTL per data type (plus 7-day stale
      mirrors served when the upstream circuit breaker is open)
- [x] Design canonical data models: `Team`, `Player`, `GameResult`, `Schedule`, `InjuryReport` (sport-agnostic base
      types matching the OpenAPI schemas; deterministic UUIDv5 ids derived from `league:kind:external_id`)
- [x] Implement NBA adapter: fetch team stats, player stats, game logs, schedules, and game results from
      NBA.com HTTP endpoints directly in Go (per [ADR-020](../../decisions/020-statistics-data-bridge.md));
      injury reports come from ESPN's public API (ADR-008 fallback — stats.nba.com has no injury endpoint),
      flag-gated via `INJURY_SOURCE=espn|none` and degrading to empty reports on failure
- [x] Implement stats normalization: convert NBA-specific fields into canonical format
- [x] Implement derived statistics computation: rolling averages (last N games via `LastNGames`),
      offensive/defensive ratings, pace, efficiency, per-game rates (upstream Advanced values authoritative,
      standard formulas as the fallback, golden-tested to agree within tolerance)
- [x] Build REST API endpoints (the documented Phase 1 set — 11 operations):
  - [x] `GET /api/v1/stats/teams?league={league}` -- all teams for a league (plus `/teams/{team_id}` detail)
  - [x] `GET /api/v1/stats/teams/{team_id}/stats` -- single team stats (stat_type shaping, rolling windows, splits)
  - [x] `GET /api/v1/stats/players/{player_id}` -- single player stats (plus `/players` list and optional game log)
  - [x] `GET /api/v1/stats/games?league={league}` -- game results (plus `/games/{game_id}` and
        `/games/{game_id}/result` for bet grading)
  - [x] `GET /api/v1/stats/schedule?league={league}` -- upcoming games
  - [x] `GET /api/v1/stats/injuries?league={league}` -- injury reports
- [x] Implement Redis pub/sub: publish `stats.updated` (on change, with injury status diffs) and `game.completed`
      (exactly once per game via a restart-safe dedup marker, detected by a 5-minute scoreboard watcher that idles
      when no games are pending)
- [x] Add OpenAPI spec (hand-authored in bookie-breaker-docs `api-contracts/openapi/statistics-service.yaml`,
      spec-first per [ADR-021](../../decisions/021-openapi-spec-strategy.md))
- [x] Write unit tests for normalization and derived stat computation
- [x] Write integration tests against real Redis and Postgres (testcontainers; NBA/ESPN stubbed with fixtures)
- [x] Create Dockerfile (multi-stage build) and `.config/air.toml` for hot reload
- [x] Add `.env.example` with all service-specific environment variables

**Phase 6 additions (sport expansion):**

- [ ] Implement NFL adapter using nfl_data_py
- [ ] Implement MLB adapter using pybaseball
- [ ] Implement NCAA Basketball adapter
- [ ] Implement NCAA Football adapter using CFBD API
- [ ] Implement NCAA Baseball adapter

**Deferred (revisit when a concrete consumer appears):**

- A dedicated free-text search endpoint (`GET /api/v1/stats/search?q=...&type=team|player|game`) was considered
  during the 2026-07-03 Phase 1 review and deferred. The list endpoints already expose rich filters in the OpenAPI
  spec (teams: league/season/conference/division/active; players: team_id/league/position/status; games:
  league/season/date range/team/status; schedule: league/date range/team_id), which covers programmatic lookup.
  Revisit when the CLI or UI needs autocomplete-style search.
- `GET /api/v1/stats/games/{game_id}/box-score` and `GET /api/v1/stats/matchup/{home}/{away}` exist in the OpenAPI
  contract but their consumers arrive later (box scores: Phase 7 prop grading and model training; matchup context:
  Phase 2+ prediction features). Implement them alongside those consumers.
- `GET /api/v1/stats/venues/{venue_id}` and `GET /api/v1/stats/leagues` exist in the OpenAPI contract but appear in
  no phase plan (contract-only, recorded during the 2026-07-03 Phase 1 build). Venue coordinates become relevant for
  Phase 2 travel-distance features; `/leagues` is essentially static config. Decide their phase when a consumer
  appears.
- Historical seasons are not cached: `season` params other than the current season return an error/empty results.
  Revisit when model training (Phase 2) needs historical game logs.

**Implementation notes (2026-07-03):**

- Cache values are stored as String(JSON) rather than the Hash types sketched in
  `schemas/database-schemas/redis-schemas.md`, for consistency with lines-service — reconcile the schema doc.
- `wins`/`losses` (TeamStats), `external_id` (Game), and `league` (PlayerSummary) are additive response fields not in
  the OpenAPI schemas; they must serialize because the Redis cache is the service's primary store.

## Dependencies

- **infra-ops** must provide Docker Compose with Redis running
- No other service dependencies (statistics-service has no upstream services)

## Complexity

**XL** -- Multiple external API integrations with different data formats, derived stat computation, caching strategy,
and eventual 6-sport support. The Go ↔ Python bridge for sport data packages (nba_api, nfl_data_py, pybaseball) adds
architectural complexity.

## Definition of Done

- [ ] `GET /api/v1/stats/teams?league=nba` returns current NBA team statistics (code-complete and integration-tested
      against fixtures; needs a full-stack run against live stats.nba.com — see the Phase 1 status note)
- [ ] `GET /api/v1/stats/schedule?league=nba` returns upcoming NBA games (same caveat as above)
- [ ] `GET /api/v1/stats/games?league=nba` returns completed game results with scores (same caveat as above)
- [ ] Stats responses are cached in Redis (cache hit returns in < 10ms — latency measured at E2E verification)
- [x] Derived statistics (rolling averages, ratings) are computed correctly (golden-tested against upstream values)
- [x] `stats.updated` and `game.completed` events are published to Redis (integration-tested, exactly-once)
- [x] Health check endpoint responds with service status
- [x] OpenAPI spec is generated and valid
- [x] Unit and integration tests pass
- [ ] Service starts cleanly in Docker Compose (compose blocks added; full `task up` verification pending E2E)

## Key Documentation

- [Statistics Service Component](../bookie-breaker-docs/components/statistics-service.md)
- [Statistics Data Sources (ADR-008)](../bookie-breaker-docs/decisions/008-statistics-data-sources.md)
- [Data Flow Architecture](../bookie-breaker-docs/architecture/data-flow.md)
- [Feature Inventory: PIPE-001 through PIPE-007](../bookie-breaker-docs/architecture/feature-inventory.md)
