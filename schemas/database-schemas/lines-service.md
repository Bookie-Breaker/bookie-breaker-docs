# Lines Service Database Schema

**Schema name:** `lines`
**Owner service:** lines-service (Go)
**Storage type:** PostgreSQL 16 + TimescaleDB

---

## Overview

The lines schema stores all betting line data ingested from external odds APIs. The primary table `line_snapshots` is a TimescaleDB hypertable partitioned by `captured_at`, enabling efficient time-series queries, automatic compression of historical data, and retention policies to manage storage growth.

Estimated volume: **5-10 million rows/year** across all leagues, growing at ~15,000-30,000 rows/day during peak multi-sport overlap (October-March).

---

## Schema Setup

```sql
CREATE SCHEMA IF NOT EXISTS lines;
SET search_path TO lines;

-- Enable TimescaleDB (instance-level, run once)
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

---

## Tables

### sportsbooks

Canonical registry of tracked sportsbooks. Near-static reference data (~50 rows).

```sql
CREATE TABLE lines.sportsbooks (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    key         TEXT NOT NULL UNIQUE,
    is_sharp    BOOLEAN NOT NULL DEFAULT FALSE,
    is_active   BOOLEAN NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

COMMENT ON TABLE lines.sportsbooks IS 'Canonical sportsbook registry. ~50 rows, near-static.';
COMMENT ON COLUMN lines.sportsbooks.key IS 'Unique slug used in external APIs (e.g., "draftkings", "pinnacle").';
COMMENT ON COLUMN lines.sportsbooks.is_sharp IS 'Market-making books (Pinnacle, Circa) whose lines are considered efficient.';

-- Indexes
CREATE INDEX idx_sportsbooks_active ON lines.sportsbooks (is_active) WHERE is_active = TRUE;
```

### line_snapshots (TimescaleDB Hypertable)

Immutable, append-only record of every betting line snapshot captured from external APIs. This is the primary table and the highest-volume table in the system.

Each row represents a single line offered by a single sportsbook for a single market on a single game at a specific point in time. Lines are never updated -- a new row is inserted when a line changes.

```sql
CREATE TABLE lines.line_snapshots (
    id               UUID NOT NULL DEFAULT gen_random_uuid(),
    game_external_id TEXT NOT NULL,
    sportsbook_id    UUID NOT NULL REFERENCES lines.sportsbooks(id),
    league           league_enum NOT NULL,
    market_type      market_type_enum NOT NULL,
    selection        TEXT NOT NULL,
    line_value       DECIMAL(8,2),
    odds_american    INTEGER NOT NULL,
    odds_decimal     DECIMAL(8,4) NOT NULL,
    is_live          BOOLEAN NOT NULL DEFAULT FALSE,
    captured_at      TIMESTAMPTZ NOT NULL,
    source           TEXT NOT NULL,

    -- TimescaleDB requires the partition column in the PK
    PRIMARY KEY (id, captured_at)
);

COMMENT ON TABLE lines.line_snapshots IS 'Immutable line snapshots. TimescaleDB hypertable partitioned by captured_at. ~5-10M rows/year.';
COMMENT ON COLUMN lines.line_snapshots.game_external_id IS 'External game identifier from the odds API. Resolved to internal game UUID by consumers via statistics-service.';
COMMENT ON COLUMN lines.line_snapshots.selection IS 'Human-readable selection string (e.g., "KC -3.5", "Over 47.5", "P.Mahomes Over 285.5 Pass Yds").';
COMMENT ON COLUMN lines.line_snapshots.line_value IS 'Spread, total, or prop number. NULL for moneylines.';
COMMENT ON COLUMN lines.line_snapshots.odds_american IS 'American odds format (e.g., -110, +150). Canonical format.';
COMMENT ON COLUMN lines.line_snapshots.odds_decimal IS 'Decimal odds (e.g., 1.91, 2.50). Denormalized for convenience.';
COMMENT ON COLUMN lines.line_snapshots.source IS 'Which API provided this line (e.g., "the_odds_api", "sharp_api").';

-- Convert to TimescaleDB hypertable
SELECT create_hypertable(
    'lines.line_snapshots',
    by_range('captured_at', INTERVAL '1 day')
);

-- Composite unique constraint for deduplication during ingestion
CREATE UNIQUE INDEX uq_line_snapshots_composite
    ON lines.line_snapshots (game_external_id, sportsbook_id, market_type, selection, captured_at);

-- Primary query pattern: current/historical lines for a game + market
CREATE INDEX idx_line_snapshots_game_market
    ON lines.line_snapshots (game_external_id, market_type, sportsbook_id, captured_at DESC);

-- League-scoped queries (e.g., "all NFL lines today")
CREATE INDEX idx_line_snapshots_league_time
    ON lines.line_snapshots (league, captured_at DESC);

-- Sportsbook-level queries
CREATE INDEX idx_line_snapshots_sportsbook
    ON lines.line_snapshots (sportsbook_id, market_type, captured_at DESC);

-- Live line filtering
CREATE INDEX idx_line_snapshots_live
    ON lines.line_snapshots (game_external_id, captured_at DESC)
    WHERE is_live = TRUE;
```

### closing_lines

Materialized closing lines -- the final line snapshot before game start for each game/sportsbook/market/selection combination. Populated by a background job when a game transitions to IN_PROGRESS or FINAL.

```sql
CREATE TABLE lines.closing_lines (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    game_external_id TEXT NOT NULL,
    sportsbook_id    UUID NOT NULL REFERENCES lines.sportsbooks(id),
    league           league_enum NOT NULL,
    market_type      market_type_enum NOT NULL,
    selection        TEXT NOT NULL,
    line_value       DECIMAL(8,2),
    odds_american    INTEGER NOT NULL,
    odds_decimal     DECIMAL(8,4) NOT NULL,
    captured_at      TIMESTAMPTZ NOT NULL,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_closing_lines_composite
        UNIQUE (game_external_id, sportsbook_id, market_type, selection)
);

COMMENT ON TABLE lines.closing_lines IS 'Materialized closing lines for CLV calculations. One row per game/sportsbook/market/selection. ~100K rows/year.';

-- Primary lookup: closing line for a specific game
CREATE INDEX idx_closing_lines_game
    ON lines.closing_lines (game_external_id, market_type, sportsbook_id);

-- League-scoped queries
CREATE INDEX idx_closing_lines_league
    ON lines.closing_lines (league, created_at DESC);
```

---

## TimescaleDB Configuration

### Chunk Interval

```sql
-- 1-day chunks. Each chunk covers 24 hours of line_snapshots.
-- At ~15K-30K rows/day, each chunk is ~5-10 MB uncompressed.
-- This provides good query performance for time-range scans.
SELECT set_chunk_time_interval('lines.line_snapshots', INTERVAL '1 day');
```

### Compression Policy

```sql
-- Compress chunks older than 7 days.
-- Compressed chunks use ~80-90% less storage.
-- Compressed data is still queryable but inserts/updates are not allowed.
ALTER TABLE lines.line_snapshots SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'game_external_id, sportsbook_id',
    timescaledb.compress_orderby = 'captured_at DESC'
);

SELECT add_compression_policy('lines.line_snapshots', INTERVAL '7 days');
```

**Segment-by columns:** `game_external_id, sportsbook_id` -- most queries filter by game and optionally by sportsbook, so segmenting by these columns allows the decompressor to read only the relevant segments.

**Order-by column:** `captured_at DESC` -- queries typically want the most recent snapshots first.

### Retention Policy

```sql
-- Retain raw (uncompressed) data for 7 days, compressed data for 2 full seasons.
-- "2 seasons" approximates to ~18 months to cover any sport's full cycle.
-- After 18 months, data is dropped. Closing lines (in the closing_lines table) are retained indefinitely.
SELECT add_retention_policy('lines.line_snapshots', INTERVAL '18 months');
```

**Rationale:** Granular line movement data is most valuable for the current season and the prior season (for model training). Beyond that, only closing lines matter for historical CLV analysis. The `closing_lines` table preserves closing data indefinitely.

---

## Continuous Aggregates

Pre-computed materialized views for line movement queries at different time granularities. These power the line movement charts in the UI and the line movement features in the prediction-engine.

### 5-Minute Buckets

```sql
CREATE MATERIALIZED VIEW lines.line_movement_5min
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('5 minutes', captured_at) AS bucket,
    game_external_id,
    sportsbook_id,
    market_type,
    selection,
    LAST(line_value, captured_at) AS line_value,
    LAST(odds_american, captured_at) AS odds_american,
    LAST(odds_decimal, captured_at) AS odds_decimal,
    COUNT(*) AS snapshot_count
FROM lines.line_snapshots
GROUP BY bucket, game_external_id, sportsbook_id, market_type, selection
WITH NO DATA;

-- Refresh policy: keep 5-min aggregates up to date, retain for 7 days
SELECT add_continuous_aggregate_policy('lines.line_movement_5min',
    start_offset    => INTERVAL '1 hour',
    end_offset      => INTERVAL '5 minutes',
    schedule_interval => INTERVAL '5 minutes'
);

SELECT add_retention_policy('lines.line_movement_5min', INTERVAL '7 days');
```

### 1-Hour Buckets

```sql
CREATE MATERIALIZED VIEW lines.line_movement_1hr
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', captured_at) AS bucket,
    game_external_id,
    sportsbook_id,
    market_type,
    selection,
    FIRST(line_value, captured_at) AS open_line_value,
    LAST(line_value, captured_at) AS close_line_value,
    FIRST(odds_american, captured_at) AS open_odds,
    LAST(odds_american, captured_at) AS close_odds,
    MIN(odds_american) AS min_odds,
    MAX(odds_american) AS max_odds,
    COUNT(*) AS snapshot_count
FROM lines.line_snapshots
GROUP BY bucket, game_external_id, sportsbook_id, market_type, selection
WITH NO DATA;

SELECT add_continuous_aggregate_policy('lines.line_movement_1hr',
    start_offset    => INTERVAL '4 hours',
    end_offset      => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

SELECT add_retention_policy('lines.line_movement_1hr', INTERVAL '3 months');
```

### 1-Day Buckets

```sql
CREATE MATERIALIZED VIEW lines.line_movement_1day
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', captured_at) AS bucket,
    game_external_id,
    sportsbook_id,
    market_type,
    selection,
    league,
    FIRST(line_value, captured_at) AS open_line_value,
    LAST(line_value, captured_at) AS close_line_value,
    FIRST(odds_american, captured_at) AS open_odds,
    LAST(odds_american, captured_at) AS close_odds,
    MIN(odds_american) AS min_odds,
    MAX(odds_american) AS max_odds,
    COUNT(*) AS snapshot_count
FROM lines.line_snapshots
GROUP BY bucket, game_external_id, sportsbook_id, market_type, selection, league
WITH NO DATA;

SELECT add_continuous_aggregate_policy('lines.line_movement_1day',
    start_offset    => INTERVAL '3 days',
    end_offset      => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);

-- Retain daily aggregates for the full retention period
SELECT add_retention_policy('lines.line_movement_1day', INTERVAL '18 months');
```

---

## Key Query Patterns

| Query                                   | Table/View           | Index Used                           |
| --------------------------------------- | -------------------- | ------------------------------------ |
| Current lines for a game                | `line_snapshots`     | `idx_line_snapshots_game_market`     |
| Line movement for a game (last 24h)     | `line_movement_5min` | Hypertable chunk exclusion + segment |
| Line movement for a game (last week)    | `line_movement_1hr`  | Hypertable chunk exclusion + segment |
| Line movement for a game (full history) | `line_movement_1day` | Hypertable chunk exclusion + segment |
| Closing lines for CLV                   | `closing_lines`      | `idx_closing_lines_game`             |
| All lines for a league today            | `line_snapshots`     | `idx_line_snapshots_league_time`     |
| Best line across sportsbooks            | `line_snapshots`     | `idx_line_snapshots_game_market`     |
| Deduplication check during ingestion    | `line_snapshots`     | `uq_line_snapshots_composite`        |

---

## Volume Estimates

| Metric                       | Value             |
| ---------------------------- | ----------------- |
| Rows/year (line_snapshots)   | 5-10 million      |
| Rows/day (peak season)       | 15,000-30,000     |
| Rows/day (off-season)        | ~1,000-5,000      |
| Uncompressed row size        | ~200-300 bytes    |
| Uncompressed table size/year | 2-5 GB            |
| Compressed table size/year   | ~300-600 MB       |
| Closing lines rows/year      | ~100,000          |
| Sportsbooks rows             | ~50 (near-static) |
