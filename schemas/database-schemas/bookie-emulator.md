# Bookie Emulator Database Schema

**Schema name:** `emulator`
**Owner service:** bookie-emulator (Python/FastAPI)
**Storage type:** Standard PostgreSQL 16 tables

---

## Overview

The emulator schema stores all paper trading data: virtual bets, grading results, bankroll history, and pre-computed performance summaries. Volume is modest compared to the lines and predictions schemas -- the system places at most ~50 bets/day.

Estimated volume: **5,000-20,000 bets/year**, with proportional grades, snapshots, and summaries.

---

## Schema Setup

```sql
CREATE SCHEMA IF NOT EXISTS emulator;
SET search_path TO emulator;
```

---

## Tables

### paper_bets

Virtual bets placed when the system detects an edge. Each row captures the full context at placement time -- odds, stake, edge size, and the agent's reasoning. Bets are created with status `OPEN` and updated to a terminal status when graded.

```sql
CREATE TABLE emulator.paper_bets (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    game_external_id     TEXT NOT NULL,
    league               league_enum NOT NULL,
    market_type          market_type_enum NOT NULL,
    selection            TEXT NOT NULL,
    line_value           DECIMAL(8,2),
    odds_american        INTEGER NOT NULL,
    odds_decimal         DECIMAL(8,4) NOT NULL,
    stake                DECIMAL(10,4) NOT NULL,
    edge_at_placement    DECIMAL(6,5) NOT NULL,
    kelly_fraction       DECIMAL(6,5) NOT NULL,
    reasoning            TEXT,
    prediction_id        UUID,
    status               bet_result_enum NOT NULL DEFAULT 'OPEN',
    placed_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    graded_at            TIMESTAMPTZ,

    CONSTRAINT chk_paper_bets_stake_positive
        CHECK (stake > 0),
    CONSTRAINT chk_paper_bets_edge_positive
        CHECK (edge_at_placement > 0),
    CONSTRAINT chk_paper_bets_kelly_range
        CHECK (kelly_fraction >= 0 AND kelly_fraction <= 1)
);

COMMENT ON TABLE emulator.paper_bets IS 'Paper bets placed by the system. ~5K-20K rows/year.';
COMMENT ON COLUMN emulator.paper_bets.game_external_id IS 'External game identifier, matched against statistics-service for grading.';
COMMENT ON COLUMN emulator.paper_bets.selection IS 'Human-readable bet selection (e.g., "KC -3.5", "Over 47.5").';
COMMENT ON COLUMN emulator.paper_bets.line_value IS 'Spread, total, or prop number at placement. NULL for moneylines.';
COMMENT ON COLUMN emulator.paper_bets.odds_american IS 'American odds captured at time of placement.';
COMMENT ON COLUMN emulator.paper_bets.odds_decimal IS 'Decimal odds at placement. Used for P/L calculation.';
COMMENT ON COLUMN emulator.paper_bets.stake IS 'Stake in units (e.g., 1.0 = 1 unit).';
COMMENT ON COLUMN emulator.paper_bets.edge_at_placement IS 'Edge size (predicted_prob - implied_prob) at time of placement.';
COMMENT ON COLUMN emulator.paper_bets.kelly_fraction IS 'Kelly criterion fraction used for stake sizing (e.g., 0.25 = quarter Kelly).';
COMMENT ON COLUMN emulator.paper_bets.reasoning IS 'Agent-generated explanation of why this bet was placed.';
COMMENT ON COLUMN emulator.paper_bets.prediction_id IS 'Reference to the prediction that identified the edge. UUID from prediction-engine, not a FK (cross-service).';
COMMENT ON COLUMN emulator.paper_bets.status IS 'Bet lifecycle: OPEN -> WON/LOST/PUSH/VOID.';

-- Open bets for grading (partial index for efficiency)
CREATE INDEX idx_paper_bets_open
    ON emulator.paper_bets (game_external_id)
    WHERE status = 'OPEN';

-- Chronological bet ledger
CREATE INDEX idx_paper_bets_placed
    ON emulator.paper_bets (placed_at DESC);

-- Performance breakdown queries
CREATE INDEX idx_paper_bets_league_market
    ON emulator.paper_bets (league, market_type, status);

-- Game-based lookup
CREATE INDEX idx_paper_bets_game
    ON emulator.paper_bets (game_external_id, status);
```

### bet_grades

Grading results for completed bets. One-to-one relationship with paper_bets. Created when a game completes and the bet is graded.

```sql
CREATE TABLE emulator.bet_grades (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bet_id              UUID NOT NULL UNIQUE REFERENCES emulator.paper_bets(id) ON DELETE CASCADE,
    actual_result       TEXT NOT NULL,
    profit_loss         DECIMAL(10,4) NOT NULL,
    closing_line_value  DECIMAL(8,2),
    closing_odds        INTEGER,
    clv                 DECIMAL(6,5),
    graded_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

COMMENT ON TABLE emulator.bet_grades IS 'Grading results for completed bets. One per graded bet. ~5K-20K rows/year.';
COMMENT ON COLUMN emulator.bet_grades.actual_result IS 'Human-readable result description (e.g., "KC won by 7, covering -3.5").';
COMMENT ON COLUMN emulator.bet_grades.profit_loss IS 'P/L in units. Win: stake * (odds_decimal - 1). Loss: -stake. Push: 0.';
COMMENT ON COLUMN emulator.bet_grades.closing_line_value IS 'Closing line value from lines-service (spread/total number at close).';
COMMENT ON COLUMN emulator.bet_grades.closing_odds IS 'Closing American odds from lines-service.';
COMMENT ON COLUMN emulator.bet_grades.clv IS 'Closing Line Value: implied_prob(closing) - implied_prob(placement). Positive = captured value before market corrected.';

-- Lookup by bet
CREATE INDEX idx_bet_grades_bet
    ON emulator.bet_grades (bet_id);

-- Time-range queries for performance analysis
CREATE INDEX idx_bet_grades_graded
    ON emulator.bet_grades (graded_at DESC);
```

### bankroll_snapshots

Point-in-time bankroll state for performance trend charting. Snapshots are taken after each bet is graded and at end-of-day.

```sql
CREATE TABLE emulator.bankroll_snapshots (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    balance           DECIMAL(12,4) NOT NULL,
    total_wagered     DECIMAL(12,4) NOT NULL DEFAULT 0,
    total_profit_loss DECIMAL(12,4) NOT NULL DEFAULT 0,
    open_bets_count   INTEGER NOT NULL DEFAULT 0,
    snapshot_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

COMMENT ON TABLE emulator.bankroll_snapshots IS 'Point-in-time bankroll state. Taken after each grading + end-of-day. ~1K-5K rows/year.';
COMMENT ON COLUMN emulator.bankroll_snapshots.balance IS 'Current bankroll balance in units.';
COMMENT ON COLUMN emulator.bankroll_snapshots.total_wagered IS 'Cumulative total units wagered across all bets.';
COMMENT ON COLUMN emulator.bankroll_snapshots.total_profit_loss IS 'Cumulative net P/L in units.';
COMMENT ON COLUMN emulator.bankroll_snapshots.open_bets_count IS 'Number of currently open (ungraded) bets at snapshot time.';

-- Time-series queries for bankroll charting
CREATE INDEX idx_bankroll_snapshots_time
    ON emulator.bankroll_snapshots (snapshot_at DESC);
```

### performance_summaries

Pre-computed performance aggregates by dimension (league, market_type, time period). Updated after each bet is graded. Avoids expensive aggregation queries on the paper_bets table.

```sql
CREATE TABLE emulator.performance_summaries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dimension       TEXT NOT NULL,
    dimension_value TEXT NOT NULL,
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ NOT NULL,
    total_bets      INTEGER NOT NULL DEFAULT 0,
    wins            INTEGER NOT NULL DEFAULT 0,
    losses          INTEGER NOT NULL DEFAULT 0,
    pushes          INTEGER NOT NULL DEFAULT 0,
    total_wagered   DECIMAL(12,4) NOT NULL DEFAULT 0,
    total_profit    DECIMAL(12,4) NOT NULL DEFAULT 0,
    roi             DECIMAL(8,5) NOT NULL DEFAULT 0,
    avg_clv         DECIMAL(6,5),
    avg_edge        DECIMAL(6,5),
    brier_score     DECIMAL(6,5),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_performance_summaries_dimension_period
        UNIQUE (dimension, dimension_value, period_start, period_end)
);

COMMENT ON TABLE emulator.performance_summaries IS 'Pre-computed performance by dimension. Updated after each grading.';
COMMENT ON COLUMN emulator.performance_summaries.dimension IS 'Aggregation dimension: "league", "market_type", "overall", "monthly".';
COMMENT ON COLUMN emulator.performance_summaries.dimension_value IS 'Dimension value: "NFL", "SPREAD", "all", "2026-03".';
COMMENT ON COLUMN emulator.performance_summaries.roi IS 'Return on investment: total_profit / total_wagered.';
COMMENT ON COLUMN emulator.performance_summaries.avg_clv IS 'Average closing line value across graded bets in this dimension.';
COMMENT ON COLUMN emulator.performance_summaries.brier_score IS 'Brier score for probability calibration assessment.';

-- Lookup by dimension
CREATE INDEX idx_performance_summaries_dimension
    ON emulator.performance_summaries (dimension, dimension_value, period_start DESC);

-- Time-range queries
CREATE INDEX idx_performance_summaries_period
    ON emulator.performance_summaries (period_start DESC, period_end);
```

---

## Key Query Patterns

| Query                            | Table                       | Index Used                              |
| -------------------------------- | --------------------------- | --------------------------------------- |
| Open bets for grading            | `paper_bets`                | `idx_paper_bets_open`                   |
| Bet ledger (chronological)       | `paper_bets`                | `idx_paper_bets_placed`                 |
| Bets for a specific game         | `paper_bets`                | `idx_paper_bets_game`                   |
| Performance by league + market   | `paper_bets`                | `idx_paper_bets_league_market`          |
| Grade details for a bet          | `bet_grades`                | `idx_bet_grades_bet`                    |
| Bankroll history for charting    | `bankroll_snapshots`        | `idx_bankroll_snapshots_time`           |
| Performance summary by dimension | `performance_summaries`     | `idx_performance_summaries_dimension`   |
| Calibration data (bucketed)      | `paper_bets` + `bet_grades` | Join on bet_id, aggregated in app layer |

---

## Volume Estimates

| Metric                  | Value                                |
| ----------------------- | ------------------------------------ |
| Paper bets/year         | 5,000-20,000                         |
| Paper bets/day          | 5-50                                 |
| Bet grades/year         | 5,000-20,000 (1:1 with graded bets)  |
| Bankroll snapshots/year | 1,000-5,000                          |
| Performance summaries   | ~200-500 rows (dimensions x periods) |
| Paper bet row size      | ~500 bytes                           |
| Bet grade row size      | ~200 bytes                           |
| Total storage/year      | ~20-50 MB                            |

---

## Example Queries

### Get all open bets for a completed game

```sql
SELECT pb.*
FROM emulator.paper_bets pb
WHERE pb.game_external_id = 'odds_api_abc123'
  AND pb.status = 'OPEN';
```

### Grade a bet and insert result

```sql
-- Update bet status
UPDATE emulator.paper_bets
SET status = 'WON',
    graded_at = NOW()
WHERE id = 'bet-uuid-here';

-- Insert grade
INSERT INTO emulator.bet_grades (bet_id, actual_result, profit_loss, closing_line_value, closing_odds, clv)
VALUES (
    'bet-uuid-here',
    'KC won by 7, covering -3.5',
    0.9091,              -- stake * (1.9091 - 1) for -110 odds
    -3.0,                -- closing line moved to -3
    -115,                -- closing odds
    0.012                -- CLV: got better odds than close
);
```

### Get bet ledger with grades

```sql
SELECT pb.id, pb.league, pb.market_type, pb.selection, pb.odds_american,
       pb.stake, pb.edge_at_placement, pb.status, pb.placed_at,
       bg.profit_loss, bg.clv, bg.actual_result
FROM emulator.paper_bets pb
LEFT JOIN emulator.bet_grades bg ON bg.bet_id = pb.id
WHERE pb.league = 'NFL'
ORDER BY pb.placed_at DESC
LIMIT 50;
```

### Get ROI by league

```sql
SELECT dimension_value AS league, total_bets, wins, losses,
       roi, avg_clv, brier_score
FROM emulator.performance_summaries
WHERE dimension = 'league'
  AND period_start = '2025-09-01'
  AND period_end = '2026-02-15'
ORDER BY roi DESC;
```

### Bankroll trend for charting

```sql
SELECT snapshot_at, balance, total_profit_loss, open_bets_count
FROM emulator.bankroll_snapshots
WHERE snapshot_at >= NOW() - INTERVAL '30 days'
ORDER BY snapshot_at ASC;
```
