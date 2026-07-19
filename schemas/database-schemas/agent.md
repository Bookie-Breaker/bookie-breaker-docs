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
COMMENT ON COLUMN agent.pipeline_runs.trigger IS 'What started the run: MANUAL (API), SCHEDULED (cron scheduler), EVENT (debounced lines/stats reaction).';
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

> **Phase 7 (migration 0007):** the side vocabulary gained `YES`/`NO` (single-sided props), the structured prop
> columns from [ADR-029](../../decisions/029-prop-line-representation.md) were added alongside `selection`, and
> `is_live` flags edges detected against in-game lines.

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
    player_external_id    TEXT,
    stat_type             TEXT,
    prop_type             TEXT,
    is_live               BOOLEAN NOT NULL DEFAULT FALSE,
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
        CHECK (side IS NULL OR side IN ('HOME', 'AWAY', 'DRAW', 'OVER', 'UNDER', 'YES', 'NO')),
    CONSTRAINT chk_edges_predicted_probability_range
        CHECK (predicted_probability > 0 AND predicted_probability < 1),
    CONSTRAINT chk_edges_implied_probability_range
        CHECK (implied_probability > 0 AND implied_probability < 1)
);

COMMENT ON TABLE agent.edges IS 'Point-in-time edge detections. ~10K-50K rows/year.';
COMMENT ON COLUMN agent.edges.game_id IS 'Game identifier from statistics-service (deterministic UUIDv5). Cross-service, no FK.';
COMMENT ON COLUMN agent.edges.game_external_id IS 'Game identifier in the lines-service id space (The Odds API event id).';
COMMENT ON COLUMN agent.edges.selection IS 'Human-readable selection (e.g., "LAL -3.5", "Over 224.5").';
COMMENT ON COLUMN agent.edges.side IS 'Machine-readable side: HOME/AWAY for spreads and moneylines, DRAW for three-way moneylines (ADR-027), OVER/UNDER for totals and over/under props, YES/NO for single-sided props (ADR-029).';
COMMENT ON COLUMN agent.edges.player_external_id IS 'External player identifier for PLAYER_PROP edges. NULL for non-player markets (ADR-029).';
COMMENT ON COLUMN agent.edges.stat_type IS 'Normalized stat category for prop edges (the raw Odds API market key). NULL for non-prop markets (ADR-029).';
COMMENT ON COLUMN agent.edges.prop_type IS 'Prop structure ("OVER_UNDER" or "YES_NO"). NULL for non-prop markets (ADR-029).';
COMMENT ON COLUMN agent.edges.is_live IS 'TRUE for edges detected against in-game (live) lines; drives the live re-evaluation branch.';
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

## Phase 4 Tables (migration 0002)

### analyses

Persisted LLM output with token accounting (which covers the deferred `query_log`'s cost-accounting purpose).

```sql
CREATE TABLE agent.analyses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_type TEXT NOT NULL
        CHECK (analysis_type IN ('GAME_PREVIEW', 'EDGE_BREAKDOWN', 'PERFORMANCE_REVIEW', 'DAILY_SUMMARY')),
    game_id UUID,
    edge_id UUID REFERENCES agent.edges(id),
    title TEXT NOT NULL,
    content TEXT NOT NULL,          -- markdown
    question TEXT,                  -- operator's free-form question, when given
    model_used TEXT NOT NULL,
    provider TEXT NOT NULL,         -- anthropic | ollama
    input_summary TEXT,
    input_tokens INTEGER,
    output_tokens INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_analyses_game_type ON agent.analyses (game_id, analysis_type);
```

### edge_alerts

Alert delivery tracking: one row per `events:edge.detected` publish, carrying the natural-language
description and the full event payload, plus acknowledgement state for `PUT /alerts/{id}/acknowledge`.

```sql
CREATE TABLE agent.edge_alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    edge_id UUID NOT NULL REFERENCES agent.edges(id),
    channel TEXT NOT NULL DEFAULT 'redis' CHECK (channel IN ('redis')),
    priority TEXT NOT NULL CHECK (priority IN ('LOW', 'MEDIUM', 'HIGH')),
    message TEXT NOT NULL,          -- NL description (LLM cheap tier or deterministic template)
    payload JSONB NOT NULL DEFAULT '{}',
    delivered_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    acknowledged_at TIMESTAMPTZ
);

CREATE INDEX idx_edge_alerts_delivery ON agent.edge_alerts (channel, priority, delivered_at DESC);
```

### pipeline_schedules

Source of truth for cron scheduling (ADR-015 as amended: croniter loop over this table, not APScheduler).
One row per league; `POST /schedule` upserts.

```sql
CREATE TABLE agent.pipeline_schedules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    league league_enum NOT NULL UNIQUE,
    cron_expression TEXT NOT NULL,
    timezone TEXT NOT NULL DEFAULT 'UTC',   -- IANA; cron evaluated in this zone, stored times are UTC
    description TEXT,
    enabled BOOLEAN NOT NULL DEFAULT TRUE,
    simulation_config JSONB,
    auto_bet BOOLEAN NOT NULL DEFAULT TRUE,
    min_edge_threshold NUMERIC(5, 2) NOT NULL DEFAULT 3.0,
    last_run_at TIMESTAMPTZ,
    next_run_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Phase 7 Tables (migration 0007)

Parlays persist as a parent row in `agent.parlays` with per-leg detail in `agent.parlay_legs`, mirroring the
emulator's parent + legs split ([ADR-028](../../decisions/028-parlay-data-model.md)). The parent stores the combined
odds and the correlation math that justified the parlay — the joint (correlation-adjusted) probability next to the
independence-assumption product, their difference as `correlation_edge`, and the pairwise correlation matrix as
JSONB — so later analysis can measure whether modeled correlation actually paid off.

### parlays

```sql
CREATE TABLE agent.parlays (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pipeline_run_id         UUID REFERENCES agent.pipeline_runs(id),
    league                  league_enum NOT NULL,
    combined_odds_american  INTEGER NOT NULL,
    combined_odds_decimal   DECIMAL(10,4) NOT NULL,
    joint_probability       DECIMAL(6,5) NOT NULL,
    independent_probability DECIMAL(6,5) NOT NULL,
    correlation_edge        DECIMAL(7,5) NOT NULL,
    expected_value          DECIMAL(7,5) NOT NULL,
    kelly_fraction          DECIMAL(6,5) NOT NULL,
    recommended_stake       DECIMAL(8,2) NOT NULL,
    confidence              DECIMAL(6,5),
    is_same_game            BOOLEAN NOT NULL DEFAULT FALSE,
    leg_count               INTEGER NOT NULL,
    correlations            JSONB,
    detected_at             TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at              TIMESTAMPTZ NOT NULL,
    is_stale                BOOLEAN NOT NULL DEFAULT FALSE,
    paper_bet_id            UUID,

    CONSTRAINT chk_parlays_joint_probability_range
        CHECK (joint_probability > 0 AND joint_probability < 1),
    CONSTRAINT chk_parlays_independent_probability_range
        CHECK (independent_probability > 0 AND independent_probability < 1),
    CONSTRAINT chk_parlays_leg_count
        CHECK (leg_count >= 2)
);

COMMENT ON TABLE agent.parlays IS 'Detected parlay recommendations: combined odds + correlation-adjusted probability/EV (ADR-028).';
COMMENT ON COLUMN agent.parlays.joint_probability IS 'Correlation-adjusted probability that all legs win (from the simulation joint distribution).';
COMMENT ON COLUMN agent.parlays.independent_probability IS 'Product of per-leg probabilities under the independence assumption books price with.';
COMMENT ON COLUMN agent.parlays.correlation_edge IS 'joint_probability - independent_probability: the mispricing that makes a correlated parlay attractive.';
COMMENT ON COLUMN agent.parlays.is_same_game IS 'TRUE for same-game parlays (SGPs), where leg correlation is strongest.';
COMMENT ON COLUMN agent.parlays.correlations IS 'Pairwise leg correlation matrix persisted for post-hoc analysis.';
COMMENT ON COLUMN agent.parlays.paper_bet_id IS 'Parlay parent paper-bet UUID from bookie-emulator when auto-bet placed one. Cross-service, no FK.';

-- Default listing: fresh parlays, newest first
CREATE INDEX idx_parlays_fresh
    ON agent.parlays (detected_at DESC)
    WHERE is_stale = FALSE;

-- League-filtered listings
CREATE INDEX idx_parlays_league
    ON agent.parlays (league, detected_at DESC);
```

### parlay_legs

```sql
CREATE TABLE agent.parlay_legs (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parlay_id             UUID NOT NULL REFERENCES agent.parlays(id) ON DELETE CASCADE,
    leg_index             INTEGER NOT NULL,
    game_id               UUID NOT NULL,
    game_external_id      TEXT NOT NULL,
    league                league_enum NOT NULL,
    market_type           market_type_enum NOT NULL,
    selection             TEXT NOT NULL,
    side                  TEXT,
    line_value            DECIMAL(8,2),
    player_external_id    TEXT,
    stat_type             TEXT,
    prop_type             TEXT,
    odds_american         INTEGER NOT NULL,
    odds_decimal          DECIMAL(8,4) NOT NULL,
    predicted_probability DECIMAL(6,5) NOT NULL,
    prediction_id         UUID,
    edge_id               UUID REFERENCES agent.edges(id),

    CONSTRAINT chk_parlay_legs_side
        CHECK (side IS NULL OR side IN ('HOME', 'AWAY', 'DRAW', 'OVER', 'UNDER', 'YES', 'NO')),
    CONSTRAINT uq_parlay_legs_parlay_leg_index
        UNIQUE (parlay_id, leg_index)
);

COMMENT ON TABLE agent.parlay_legs IS 'Per-leg detail for detected parlays (ADR-028). Legs reuse the single-edge field vocabulary including the ADR-029 prop columns.';
COMMENT ON COLUMN agent.parlay_legs.leg_index IS 'Position of the leg within the parlay (unique per parlay).';
COMMENT ON COLUMN agent.parlay_legs.predicted_probability IS 'Calibrated per-leg win probability used in the joint/independent computation.';
COMMENT ON COLUMN agent.parlay_legs.edge_id IS 'The single-leg edge this leg was promoted from, when applicable.';

-- Legs of a parlay
CREATE INDEX idx_parlay_legs_parlay
    ON agent.parlay_legs (parlay_id);
```

---

## Still Deferred

- **query_log** — its purpose (LLM usage accounting for ad-hoc Q&A) is covered by the token columns on
  `analyses`, and the agent has no ad-hoc query endpoint; MCP `ask_analyst` calls flow through
  `POST /analysis` and are recorded there. Revisit if a standalone query surface ever lands.

---

## Key Query Patterns

| Query                            | Table           | Index Used                        |
| -------------------------------- | --------------- | --------------------------------- |
| Current edges (fresh, newest)    | `edges`         | `idx_edges_fresh`                 |
| Edges filtered by league         | `edges`         | `idx_edges_league`                |
| Edges for a game (stale-marking) | `edges`         | `idx_edges_game_market`           |
| Current parlays (fresh, newest)  | `parlays`       | `idx_parlays_fresh`               |
| Parlays filtered by league       | `parlays`       | `idx_parlays_league`              |
| Legs of a parlay                 | `parlay_legs`   | `idx_parlay_legs_parlay`          |
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
