# Agent Database Schema

**Schema name:** `agent`
**Owner service:** agent (Python/FastAPI)
**Storage type:** Standard PostgreSQL 16 tables

---

## Overview

The agent schema persists the two entities the agent owns durably: detected edges and pipeline run records. Edges
are the agent's primary output — the comparison of calibrated probabilities against de-vigged market prices — and
pipeline runs are the audit trail of orchestration (what ran, per-step outcomes, and counters). Dashboard and slate
snapshots remain Redis-cached and transient (see [redis-schemas.md](redis-schemas.md)).

Volume is modest: one edges row per (game, market, selection, detection) and one pipeline_runs row per run.

Estimated volume: **10,000-50,000 edges/year**, **1,000-10,000 pipeline runs/year**.

> **History:** through Phase 2 the agent was documented as Redis-only. Phase 3 introduced this schema (per
> [ADR-015](../../decisions/015-pipeline-scheduler.md) and the agent component spec) so edges and run history
> survive restarts and support the `/edges` and `/pipeline/runs/{id}` contract endpoints.

---

## Schema Setup

```sql
CREATE SCHEMA IF NOT EXISTS agent;
SET search_path TO agent;
```

---

## Tables

### pipeline_runs

One row per pipeline execution (on-demand via `POST /api/v1/agent/pipeline/run` in Phase 3; event- and
schedule-triggered runs use the same table).

```sql
CREATE TABLE agent.pipeline_runs (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    league           league_enum,
    status           TEXT NOT NULL DEFAULT 'QUEUED',
    trigger          TEXT NOT NULL DEFAULT 'MANUAL',
    params           JSONB NOT NULL DEFAULT '{}',
    steps            JSONB NOT NULL DEFAULT '{}',
    games_processed  INTEGER NOT NULL DEFAULT 0,
    edges_found      INTEGER NOT NULL DEFAULT 0,
    bets_placed      INTEGER NOT NULL DEFAULT 0,
    error            TEXT,
    started_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    finished_at      TIMESTAMPTZ,

    CONSTRAINT chk_pipeline_runs_status
        CHECK (status IN ('QUEUED', 'RUNNING', 'COMPLETED', 'COMPLETED_WITH_ERRORS', 'FAILED')),
    CONSTRAINT chk_pipeline_runs_trigger
        CHECK (trigger IN ('MANUAL', 'EVENT', 'SCHEDULED'))
);

COMMENT ON TABLE agent.pipeline_runs IS 'One row per pipeline execution. ~1K-10K rows/year.';
COMMENT ON COLUMN agent.pipeline_runs.league IS 'League the run targeted. NULL = all leagues.';
COMMENT ON COLUMN agent.pipeline_runs.status IS 'Run lifecycle: QUEUED -> RUNNING -> COMPLETED / COMPLETED_WITH_ERRORS / FAILED. Agent-private vocabulary (TEXT + CHECK, not a shared enum).';
COMMENT ON COLUMN agent.pipeline_runs.trigger IS 'What started the run. Phase 3 uses MANUAL; EVENT/SCHEDULED reserved for Phase 4.';
COMMENT ON COLUMN agent.pipeline_runs.params IS 'Request parameters: game_ids, force_refresh, auto_bet, simulation_config.';
COMMENT ON COLUMN agent.pipeline_runs.steps IS 'Per-step and per-game status map (simulation/prediction/edge_detection/bet_placement with per-game errors).';
COMMENT ON COLUMN agent.pipeline_runs.error IS 'Top-level failure reason when status = FAILED.';

-- Recent runs (dashboard last_run, run listing)
CREATE INDEX idx_pipeline_runs_started
    ON agent.pipeline_runs (started_at DESC);

-- Duplicate-run guard: at most one RUNNING row per league
CREATE UNIQUE INDEX uq_pipeline_runs_running_league
    ON agent.pipeline_runs (league)
    WHERE status = 'RUNNING';
```

> The partial unique index treats NULL leagues as distinct (standard Postgres semantics), so the application layer
> additionally guards all-league runs; the index protects the common per-league case.

### edges

Persisted edge detections. Each row is a point-in-time comparison of a calibrated prediction against a de-vigged
market price at a specific book. Edges are never updated in place except for staleness marking and the paper-bet
link; a re-detection at new odds inserts a new row.

```sql
CREATE TABLE agent.edges (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pipeline_run_id       UUID REFERENCES agent.pipeline_runs(id),
    game_id               UUID NOT NULL,
    game_external_id      TEXT NOT NULL,
    league                league_enum NOT NULL,
    market_type           market_type_enum NOT NULL,
    selection             TEXT NOT NULL,
    side                  TEXT,
    line_value            DECIMAL(8,2),
    sportsbook_key        TEXT NOT NULL,
    odds_american         INTEGER NOT NULL,
    predicted_probability DECIMAL(6,5) NOT NULL,
    implied_probability   DECIMAL(6,5) NOT NULL,
    edge_percentage       DECIMAL(6,3) NOT NULL,
    expected_value        DECIMAL(7,5) NOT NULL,
    kelly_fraction        DECIMAL(6,5) NOT NULL,
    recommended_stake     DECIMAL(8,2) NOT NULL,
    confidence            DECIMAL(6,5),
    devig_method          TEXT NOT NULL DEFAULT 'multiplicative',
    prediction_id         UUID,
    simulation_run_id     UUID,
    detected_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at            TIMESTAMPTZ NOT NULL,
    is_stale              BOOLEAN NOT NULL DEFAULT FALSE,
    paper_bet_id          UUID,

    CONSTRAINT chk_edges_side
        CHECK (side IS NULL OR side IN ('HOME', 'AWAY', 'OVER', 'UNDER')),
    CONSTRAINT chk_edges_predicted_probability_range
        CHECK (predicted_probability > 0 AND predicted_probability < 1),
    CONSTRAINT chk_edges_implied_probability_range
        CHECK (implied_probability > 0 AND implied_probability < 1)
);

COMMENT ON TABLE agent.edges IS 'Point-in-time edge detections. ~10K-50K rows/year.';
COMMENT ON COLUMN agent.edges.game_id IS 'Game identifier from statistics-service (deterministic UUIDv5). Cross-service, no FK.';
COMMENT ON COLUMN agent.edges.game_external_id IS 'Game identifier in the lines-service id space (The Odds API event id).';
COMMENT ON COLUMN agent.edges.selection IS 'Human-readable selection (e.g., "LAL -3.5", "Over 224.5").';
COMMENT ON COLUMN agent.edges.side IS 'Machine-readable side: HOME/AWAY for spreads and moneylines, OVER/UNDER for totals.';
COMMENT ON COLUMN agent.edges.sportsbook_key IS 'Book offering the priced side the edge was detected at (best available price).';
COMMENT ON COLUMN agent.edges.implied_probability IS 'De-vigged implied probability (method in devig_method), not the raw implied probability.';
COMMENT ON COLUMN agent.edges.edge_percentage IS 'Edge in percentage points: (predicted - implied) * 100.';
COMMENT ON COLUMN agent.edges.expected_value IS 'EV per unit staked: predicted_prob * decimal_odds - 1.';
COMMENT ON COLUMN agent.edges.kelly_fraction IS 'Fractional-Kelly stake fraction of bankroll (quarter Kelly, capped at 5%).';
COMMENT ON COLUMN agent.edges.recommended_stake IS 'Recommended stake in units at detection time (kelly_fraction x bankroll, after exposure scaling).';
COMMENT ON COLUMN agent.edges.confidence IS 'Edge quality score in [0,1] per algorithms/edge-detection.md section 4.';
COMMENT ON COLUMN agent.edges.prediction_id IS 'Prediction UUID from prediction-engine. Cross-service, no FK.';
COMMENT ON COLUMN agent.edges.simulation_run_id IS 'Simulation run UUID from simulation-engine. Cross-service, no FK.';
COMMENT ON COLUMN agent.edges.expires_at IS 'Scheduled game start. Edges are worthless once the game begins.';
COMMENT ON COLUMN agent.edges.is_stale IS 'Set when the underlying line moved or the game completed; stale edges are excluded from default listings.';
COMMENT ON COLUMN agent.edges.paper_bet_id IS 'Paper bet UUID from bookie-emulator when auto-bet placed one. Cross-service, no FK.';

-- Game detail lookups and per-game staleness marking
CREATE INDEX idx_edges_game_market
    ON agent.edges (game_id, market_type, detected_at DESC);

-- Default listing: fresh edges, newest first
CREATE INDEX idx_edges_fresh
    ON agent.edges (detected_at DESC)
    WHERE is_stale = FALSE;

-- League-filtered listings
CREATE INDEX idx_edges_league
    ON agent.edges (league, detected_at DESC);
```

---

## Deferred to Phase 4

The agent component spec also describes `analyses`, `edge_alerts`, and `query_log` tables. These are deliberately
**not** created in Phase 3:

- **analyses / query_log** — both exist to persist LLM output and LLM usage accounting; the Anthropic/Ollama
  integration lands in Phase 4, so the tables land with it.
- **edge_alerts** — tracks alert _delivery_ (channels, acknowledgement). Phase 3 alerting is the fire-and-forget
  `events:edge.detected` publish with no delivery tracking; the table is meaningful only once Phase 4 adds
  natural-language alert routing.

APScheduler job persistence (ADR-015) also lands in Phase 4 with cron scheduling; Phase 3 runs are on-demand only.

---

## Key Query Patterns

| Query                            | Table           | Index Used                        |
| -------------------------------- | --------------- | --------------------------------- |
| Current edges (fresh, newest)    | `edges`         | `idx_edges_fresh`                 |
| Edges filtered by league         | `edges`         | `idx_edges_league`                |
| Edges for a game (stale-marking) | `edges`         | `idx_edges_game_market`           |
| Last pipeline run (dashboard)    | `pipeline_runs` | `idx_pipeline_runs_started`       |
| Duplicate-run guard              | `pipeline_runs` | `uq_pipeline_runs_running_league` |

---

## Example Queries

### Current fresh edges for the NBA

```sql
SELECT id, game_id, market_type, selection, sportsbook_key, odds_american,
       predicted_probability, implied_probability, edge_percentage,
       kelly_fraction, recommended_stake, confidence, detected_at, expires_at
FROM agent.edges
WHERE league = 'NBA'
  AND is_stale = FALSE
  AND expires_at > NOW()
ORDER BY detected_at DESC
LIMIT 50;
```

### Mark a game's edges stale after a line move

```sql
UPDATE agent.edges
SET is_stale = TRUE
WHERE game_id = 'game-uuid-here'
  AND is_stale = FALSE;
```

### Link an edge to its auto-placed paper bet

```sql
UPDATE agent.edges
SET paper_bet_id = 'paper-bet-uuid-from-emulator'
WHERE id = 'edge-uuid-here';
```
