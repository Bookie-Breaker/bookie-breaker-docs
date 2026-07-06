# ADR-027: Three-Way Markets and Regulation-Time Settlement

## Status

Accepted

## Context

Soccer ([ADR-026](026-sport-expansion-scope-and-data-sources.md)) breaks two assumptions baked into the system since
Phase 2:

1. **Every market has exactly two sides.** Soccer's primary moneyline is three-way: home win / draw / away win. The
   agent's edge detector groups lines into two-sided pairs and de-vigs exactly two probabilities; the emulator's
   grader has no DRAW selection and grades a tied moneyline as PUSH — which is wrong for three-way markets, where a
   draw _loses_ a HOME or AWAY bet and _wins_ a DRAW bet. The three-way de-vig math was already specified in
   [edge-detection.md §1](../algorithms/edge-detection.md) but never implemented.
2. **Games settle on the final score.** Soccer knockout matches proceed to extra time and penalties, but standard
   soccer markets (three-way moneyline, totals, goal lines) settle on the **90-minute regulation score**. The
   `events:game.completed` payload carries only final scores.

Options considered for the three-way representation:

- **New market type (e.g., `MONEYLINE_3WAY`)** — clean separation, but every table, index, spec, filter, and UI
  branch keyed on market_type needs a new case, and lines-service would have to classify Odds API `h2h` markets
  differently per sport.
- **MONEYLINE + a new `DRAW` side** — the market stays MONEYLINE; the third outcome is one more side value. Every
  schema already keys on (market_type, side); only side vocabularies and side-aware logic change.

Options considered for regulation settlement:

- **Separate `regulation.game.completed` event** — duplicates the event pipeline for one sport.
- **Optional regulation-score fields on the existing event** — additive, backward compatible.

## Decision

**Three-way markets are `MONEYLINE` with a third side value `DRAW`.** No new market type. Concretely:

- `BetSide` / side vocabularies / Postgres check constraints gain `DRAW` (lines-service side derivation, agent
  `edges` table, emulator `paper_bets` table, prediction-engine `predictions.side`, OpenAPI enums).
- lines-service maps The Odds API's `"Draw"` h2h outcome to side `DRAW`.
- prediction-engine emits **three prediction rows** for a three-way moneyline (sides HOME/DRAW/AWAY), sourcing the
  draw probability from the simulation output's existing `draw_probability` field; the calibrated triple is
  renormalized to sum to 1.
- The agent's edge detector groups markets as complete when their sides form `{HOME, AWAY}`, `{OVER, UNDER}`, or —
  for MONEYLINE only — `{HOME, AWAY, DRAW}`. De-vig generalizes to N outcomes (`devig_many`), implementing the
  existing spec; the two-outcome functions and their golden values are unchanged. Complement math
  (`P(other) = 1 − P`) applies only to two-sided groups.
- The emulator grades three-way moneylines with no PUSH path: margin > 0 → HOME wins, margin < 0 → AWAY wins,
  margin = 0 → DRAW wins, and the other sides lose. Two-way grading (tie → PUSH) is unchanged.

**Settlement scores are league-aware, driven by optional regulation-score fields.** `events:game.completed` (and the
game resource in the statistics-service API) gains optional `regulation_home_score` / `regulation_away_score` fields.
Absent fields mean the final score is the settlement-relevant score — every existing publisher and payload is
unchanged byte-for-byte. The soccer adapter populates them when a match goes to extra time. The emulator selects
settlement scores per league: SOCCER grades **all** markets on the regulation score (falling back to final if the
fields are absent); all other sports grade on the final score. Hockey settlement semantics (NHL standard: moneyline
and totals include overtime/shootout, shootout counts as one goal) are decided in the hockey wave; regulation-time
three-way hockey markets are deferred.

## Consequences

### Positive

- Minimal-diff: no new market type ripples through schemas, specs, generated clients, or UI filters
- Backward compatible end to end — NBA/two-way behavior, golden de-vig values, and existing event payloads are
  byte-identical
- The N-way de-vig and DRAW plumbing built for soccer is exactly what NHL regulation three-way markets and NFL's
  rare regular-season tie (grades PUSH on two-way moneylines — a case the same machinery must handle) need later

### Negative

- `side` semantics become market-dependent (DRAW is only valid on MONEYLINE), enforced by check constraints and
  validation rather than the type system
- Calibrating three outcome probabilities independently and renormalizing is not a true multinomial calibration;
  acceptable for Phase 6, to be validated against real soccer data (a multinomial head is a Phase 7+ upgrade)

### Neutral

- The Odds API also offers explicit `h2h_3_way` market keys for sports where 2-way is the default (e.g., NHL);
  adopting them later requires only normalizer mapping, not schema change
- `draw_probability` has existed in the simulation output contract since Phase 2; this ADR finally consumes it
