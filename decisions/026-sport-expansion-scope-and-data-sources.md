# ADR-026: Phase 6 Sport Expansion Scope and Data Sources

## Status

Accepted (amends [ADR-008](008-statistics-data-sources.md) and [ADR-020](020-statistics-data-bridge.md))

## Context

Phase 6 expands the system from NBA-only to all remaining leagues. During planning (July 2026), three things changed
relative to the original roadmap:

1. **Soccer was added to scope.** The 2026 FIFA World Cup (running now, through July 19) is the only major live
   competition during the summer build window, making it the ideal live test case. Soccer also introduces three-way
   (win/draw/loss) markets — see [ADR-027](027-three-way-markets-and-regulation-settlement.md).
2. **Hockey was added to scope.** The NHL has an official, free, documented REST API and its Poisson-like goal scoring
   heavily reuses the soccer simulation work. NCAA hockey is plumbed but gated on betting-line availability.
3. **The ADR-020 Python sidecar became unnecessary.** Re-evaluating sources per league showed that every league now has
   a direct REST API or static-file source that Go can consume. The packages that motivated the sidecar were wrappers
   around sources Go can reach directly: `nfl_data_py` (deprecated upstream in favor of `nflreadpy`) wraps nflverse's
   static CSV releases on GitHub; `pybaseball` wraps sources that MLB's official StatsAPI covers for our needs.

A second pattern emerged: every league needs **two data paths with different latency requirements**:

- **Real-time path** (schedules, live scores, final results → GameWatcher → `events:game.completed`): must be
  low-latency polling. Bet grading depends on it.
- **Season-aggregate path** (team stats feeding simulation and model features): refreshed a few times per day;
  latency-insensitive, so the richest source wins.

## Decision

**Scope:** league_enum grows to `NFL, NBA, MLB, NCAA_FB, NCAA_BB, NCAA_BSB, FIFA_WC, EPL, NHL, NCAA_HKY`;
sport_enum grows to `FOOTBALL, BASKETBALL, BASEBALL, SOCCER, HOCKEY`.

**Soccer is competition-config-driven:** one adapter, one simulation plugin, and one prediction model serve the
SOCCER sport. Each competition (FIFA_WC, EPL, and future additions) is a configuration entry — ESPN league code,
Odds API sport key, home-advantage setting, season shape — plus a league_enum value. The prediction model pools
training data across competitions with the competition as a feature. Club-league data does **not** directly inform
national-team strength (disjoint team pools; player-level modeling is out of scope until Phase 7+); national-team
strength comes from international match history.

**The Python sidecar is eliminated.** statistics-service remains a single Go container. Offline Python scripts for
model-training data collection remain in prediction-engine (they are development tools, not runtime services).

**Data source matrix** (supersedes the ADR-008 table):

| League   | Sport      | Real-time (watcher)                         | Season stats                                                 | Odds API sport key     |
| -------- | ---------- | ------------------------------------------- | ------------------------------------------------------------ | ---------------------- |
| NBA      | BASKETBALL | stats.nba.com (unchanged)                   | stats.nba.com                                                | basketball_nba         |
| FIFA_WC  | SOCCER     | ESPN `soccer/fifa.world`                    | ESPN standings/results                                       | soccer_fifa_world_cup  |
| EPL      | SOCCER     | ESPN `soccer/eng.1`                         | ESPN standings/results                                       | soccer_epl             |
| MLB      | BASEBALL   | MLB StatsAPI (official, free)               | MLB StatsAPI (FIP/wOBA computed from counting stats)         | baseball_mlb           |
| NCAA_BSB | BASEBALL   | ESPN `baseball/college-baseball`            | ESPN (accepted: weakest depth of all leagues)                | baseball_ncaa (sparse) |
| NFL      | FOOTBALL   | ESPN `football/nfl`                         | nflverse static CSVs (GitHub releases, incl. EPA team stats) | americanfootball_nfl   |
| NCAA_FB  | FOOTBALL   | ESPN `football/college-football`            | CFBD API (free key; stats only, never scores)                | americanfootball_ncaaf |
| NHL      | HOCKEY     | NHL API `api-web.nhle.com` (official, free) | NHL API                                                      | icehockey_nhl          |
| NCAA_HKY | HOCKEY     | ESPN `hockey/mens-college-hockey`           | ESPN                                                         | unverified — see gate  |
| NCAA_BB  | BASKETBALL | ESPN `basketball/mens-college-basketball`   | CBBD API (free key; stats only, never scores)                | basketball_ncaab       |

**ESPN's undocumented site API** (`site.api.espn.com/apis/site/v2/sports/{sport}/{league}/...`) becomes the shared
real-time source for every league except NBA (kept as-is), MLB, and NHL (official APIs are strictly better). It is
free, keyless, JSON-over-REST (no scraping), and one generalized client — the existing ESPN injuries adapter with its
path parameterized — serves all of them.

**Rate-limit rules:** CFBD and CBBD free tiers have monthly call caps. They are used only for season aggregates (a few
cached calls per day) and never polled for scores; ESPN scoreboards cover the real-time path for both NCAA leagues.

**NCAA_HKY gate:** implementation beyond enum plumbing is deferred until The Odds API is verified to carry NCAA hockey
lines (believed absent). No lines → no edges → the adapter would be dead code.

## Consequences

### Positive

- statistics-service stays one container, one language — no sidecar Dockerfile, health check, CI matrix, or
  Go↔Python runtime contract (meaningful on memory-constrained dev machines)
- MLB and NHL move to _official_ documented APIs — the two most reliable sources in the whole system
- One generalized ESPN client covers six leagues' real-time needs; golden-fixture tests amortize across all of them
- Soccer competitions beyond FIFA_WC/EPL become one-day additions (enum value + config entry)
- NCAA Baseball's source improves from R-package scraping (`baseballr` via subprocess) to plain ESPN JSON

### Negative

- ESPN's API is undocumented and can change shape without notice, and it now backs six leagues' real-time paths.
  Mitigations: golden-fixture normalizer tests, raw-response archival (`raw_api_responses` hypertable) to detect
  drift, and per-league fallbacks documented in the component specs
- nflverse CSV schemas are community-maintained and need per-season verification (Wave 3 risk)
- Computing FIP/wOBA from StatsAPI counting stats in-repo means owning published weight constants and updating them
  yearly

### Neutral

- The sidecar pattern can be revived if a future source genuinely requires a Python package with no direct
  alternative; ADR-020's analysis remains valid, its Phase 6 plan is superseded
- World Cup host home advantage (USA/MEX/CAN are not truly neutral venues) is knowingly ignored for Phase 6

## Amendment (Phase 7, 2026-07-06): soccer player props brought into scope

This ADR originally scoped soccer to team-level modeling and stated "player-level modeling is out of scope until
Phase 7+" (Decision §"Soccer is competition-config-driven"). Phase 7 planning promotes soccer **player props**
(anytime/first goalscorer, shots, shots on target, cards) into scope so the live 2026 FIFA World Cup can serve as
the test case for the Phase 7 player-prop machinery while other leagues are out of season.

What changes:

- The simulation-engine soccer plugin gains an opt-in player-distribution path: each team's simulated goal count is
  allocated across the roster by goal-share (multinomial); shots/SOT/cards are per-player Poisson draws. This reuses
  the existing team-level Dixon-Coles goal grid unchanged — player output is a strictly additive, opt-in layer (see
  the Phase 7 plan, Wave 3 / design A).
- The prediction-engine gains a SOCCER `PLAYER_PROP` model following the existing adjustment-model pattern
  (`sim_prop_probability` baseline + XGBoost adjustment + Platt + conformal).

What is **still** out of scope (unchanged from the original decision):

- **National-team player strength is not transferred from club data.** Player pools are disjoint; a national-team
  player's prop rates come from international-match history only, and are synthetic-bootstrapped where that history
  is thin. Real-data calibration is genuinely deferred to a verification session (the same posture Phase 6 took for
  team models) — accepted risk given sparse national-team data and shallow Odds API soccer-prop depth.

This amendment does not alter the team-level data-source matrix above.
