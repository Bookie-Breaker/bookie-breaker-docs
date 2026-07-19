# ADR-029: Player/Team/Game Prop Line Representation

## Status

Accepted

## Context

Phase 7 ingests prop markets (player props first: player points, shots on target, anytime goalscorer, passing yards,
etc.). Today a betting line is identified by `(game_external_id, sportsbook_id, market_type, selection, captured_at)`
where `selection` is a free-text display string. `market_type_enum` already carries `PLAYER_PROP`/`TEAM_PROP`/
`GAME_PROP`, but there is no structured way to say _which player_ and _which stat_ a prop concerns — the only carrier
is the `selection` text (e.g. `"Erling Haaland Over 2.5 Shots"`).

Consumers that need this precisely:

- **Grading** must look up a specific player's actual stat value for an exact `(player, stat_type, line_value)` tuple.
- **Edge detection** must group the OVER and UNDER of the _same_ player/stat/line to two-way de-vig them.
- **The UI/CLI** must filter and display by player and stat type.

Parsing the display string for all three is brittle: books format names and stat labels inconsistently, and
diacritics (World Cup names) make string matching fragile.

## Decision

Add **structured, nullable columns** alongside the existing `selection` string wherever a line/edge/bet is stored
(`lines.line_snapshots`, `agent.edges`, `emulator.paper_bets`, and the corresponding `parlay_legs`):

- `player_external_id TEXT NULL` — the player's external id (from the odds/stats source), null for non-player markets
- `stat_type TEXT NULL` — the canonical stat key, taken from the Odds API market key (e.g. `player_points`,
  `player_shots_on_target`, `player_goal_scorer_anytime`)
- `prop_type TEXT NULL` — `OVER_UNDER` or `YES_NO`

`selection` stays populated (it remains part of the lines uniqueness index and is the human-readable label). The new
columns are additive `ADD COLUMN`s — safe on the TimescaleDB hypertable since none are partitioning keys — plus a
partial helper index on `(game_external_id, stat_type, player_external_id, captured_at DESC)` filtered to
`market_type='PLAYER_PROP'` for the read path. The lines uniqueness index is unchanged because `selection` already
embeds player + line and therefore still disambiguates rows.

In the normalizer, prop market keys are classified by **prefix** (`player_*` → `PLAYER_PROP`, `team_*` → `TEAM_PROP`)
rather than an exhaustive per-key map, and the raw market key is stored verbatim as `stat_type`. Side comes from the
outcome name (`Over`/`Under`/`Yes` → `OVER`/`UNDER`/`YES`).

## Consequences

### Positive

- Grading, edge grouping, and filtering key on exact columns, not string parsing — robust across books and diacritics
- Additive columns + prefix classifier mean new prop types need no schema change and no per-key normalizer entry
- The domain model already anticipated this (`domain-models.md` defines `BettingLine.player_id`/`stat_type` as
  nullable), so this ADR just makes the physical schema match

### Negative

- A prop's identity is now spread across `selection` + three columns; ingestion must keep them consistent
- `player_external_id` provenance varies (Odds API supplies a `description` name, not always a stable id) — a
  name-derived key may be needed until a stable id source exists, with unmatched names logged for patching

### Neutral

- Team and game props reuse the same columns (`player_external_id` null; `stat_type` = the team/game stat)
- YES-only props (anytime goalscorer) set `prop_type=YES_NO` and `side=YES`; they cannot be two-way de-vigged and take
  a lower-confidence single-sided path in edge detection (see the Phase 7 plan / ADR-031)
