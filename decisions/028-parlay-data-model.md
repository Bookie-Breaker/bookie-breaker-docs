# ADR-028: Parlay Data Model

## Status

Accepted

## Context

Phase 7 adds parlays (multi-leg bets), including same-game parlays (SGPs) whose legs are correlated. Nothing in the
system models a bet with more than one leg: `emulator.paper_bets` is strictly one row = one selection = one game, and
the agent's edge/candidate types are single-leg. We need a data model that (a) represents a parent bet plus N legs,
(b) stores combined odds and the correlation-adjusted probability/EV that justified the bet, and (c) lets each leg be
graded independently while the parent settles only when all legs are decided.

Two shapes were considered:

1. **Encode legs as JSON on a single `paper_bets` row.** Simplest migration, but legs are not independently queryable
   or gradable, breaks the existing per-bet grade/CLV/performance machinery, and makes per-leg reconciliation opaque.
2. **Parent row + child legs table.** Mirrors how real books model parlays; each leg reuses the existing
   single-bet fields (market/side/odds/line) and grade vocabulary; the parent aggregates.

## Decision

Use a **parent + legs** model, applied consistently in the two services that own bets and edges.

**bookie-emulator (`emulator` schema):**

- `paper_bets` gains: nullable `parent_bet_id` (self-FK — unused for singles), `is_parlay BOOLEAN DEFAULT FALSE`,
  `is_live BOOLEAN DEFAULT FALSE`, and (from ADR-027's amendment) nullable `side` plus
  `player_external_id`/`stat_type`/`prop_type`. A parlay parent row holds the combined stake, combined American/decimal
  odds, correlation-adjusted `predicted_probability`, and `edge_at_placement`; its per-game columns are left null.
- New `emulator.parlay_legs`: `id`, `bet_id` (FK → `paper_bets`), `leg_index`, `game_id`, `game_external_id`,
  `league`, `market_type`, `selection`, `side`, `line_value`, `player_external_id`, `stat_type`, `prop_type`,
  `odds_american`, `odds_decimal`, and `leg_status bet_result_enum` (per-leg grade).
- `bet_grades` gains nullable `actual_stat_value NUMERIC` and `stat_type TEXT` for prop actuals.

**agent (`agent` schema):**

- New `agent.parlays` (parent): combined odds, `joint_probability`, `independent_probability`, `correlation_edge`,
  `expected_value`, `kelly_fraction`, `recommended_stake`, `confidence`, `is_same_game`, `leg_count`, `league`,
  `pipeline_run_id`, `detected_at`, `expires_at`, `is_stale`, `paper_bet_id`, and the pairwise correlation matrix as
  JSONB.
- New `agent.parlay_legs`: per-leg market/side/player/odds/`predicted_probability`, nullable `edge_id` (FK to the
  single-leg edge it was promoted from, when applicable), `leg_index`.

**Grading semantics:** a leg grades via the same paths as a single bet (`grade_bet` / `grade_player_prop`). The parent
is `WON` iff all legs win; `LOST` if any leg loses; PUSH/VOID legs drop out and the combined odds are re-priced over
the surviving legs. The `game.completed` subscriber must confirm **all** legs' games are complete before settling the
parent (a per-leg "remaining ungraded legs" guard).

All migrations follow the drop/recreate-constraint precedent set by the ADR-027 DRAW retrofit.

## Consequences

### Positive

- Legs reuse every existing single-bet field, grade path, and CLV computation; the parent is a thin aggregate
- Per-leg grading + parent re-pricing handles the real-world PUSH/VOID-leg case correctly
- Correlation inputs (`joint_probability`, `correlation_edge`, the JSONB matrix) are persisted for later analysis of
  whether modeled correlation actually paid off

### Negative

- Parlay ROI cannot be attributed to a single `market_type`; performance breakdown needs a new `bet_class` dimension
  (`single`/`parlay`/`prop`/`live`)
- Settlement timing is more complex: a parlay lingers OPEN until its slowest leg's game finishes
- `paper_bets.side`/`game_id` becoming nullable touches the most-written table — queries must be audited for non-null
  assumptions before the migration

### Neutral

- Live and prop bets reuse the same columns (`is_live`, the prop columns); a live single bet is just a `paper_bets`
  row with `is_live=true` and no parent
- The parent/legs split is symmetric across agent and emulator, keeping the edge→bet mapping straightforward
