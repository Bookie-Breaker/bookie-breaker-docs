# Playbook 06 — Seasonal Operations

Turning leagues on at season start and off at season end. BookieBreaker supports 10 leagues across 5
sports; most ship dormant because polling an off-season league wastes Odds API quota (free tier: 500
requests/month).

---

## 1. The league model

| League                 | Sport                          | Default state                    |
| ---------------------- | ------------------------------ | -------------------------------- |
| NBA, MLB, FIFA_WC, EPL | basketball / baseball / soccer | **enabled**                      |
| NFL, NCAA_FB           | football                       | dormant — enable in September    |
| NHL                    | hockey                         | dormant — enable in October      |
| NCAA_BB                | basketball                     | dormant — enable in November     |
| NCAA_BSB, NCAA_HKY     | baseball / hockey              | dormant, no seasonal runbook yet |

Two variables in `bookie-breaker-infra-ops/.env` control enablement and must stay in sync:

- `ODDS_API_SPORTS` — Odds API sport keys the lines-service polls.
- `LEAGUES_ENABLED` — leagues the statistics-service tracks.

---

## 2. Season-start checklist

Changes are **cumulative** — each month's values include everything already enabled. Edit
`bookie-breaker-infra-ops/.env`:

**September (NFL + NCAA football):**

```bash
ODDS_API_SPORTS=basketball_nba,soccer_fifa_world_cup,soccer_epl,baseball_mlb,americanfootball_nfl,americanfootball_ncaaf
LEAGUES_ENABLED=NBA,FIFA_WC,EPL,MLB,NFL,NCAA_FB
```

**October (add NHL):** append `,icehockey_nhl` and `,NHL`.

**November (add NCAA basketball):** append `,basketball_ncaab` and `,NCAA_BB`.

Then, for each newly enabled league:

```bash
task down && task up                                              # apply env changes
bb pipeline schedule set --league NFL --cron "0 10,14,18 * * *"   # enable scheduled runs
bb pipeline schedule set --league NCAA_FB --cron "0 11 * * *"
```

Verify:

```bash
bb slate --league NFL          # games appear once schedules/stats sync
bb edges --league NFL          # edges after the first pipeline run
task logs:service -- lines-service   # confirm polling for the new sport keys
```

If new enum values are rejected by Postgres after a service update, run the enum migration —
see [07 §7](07-troubleshooting.md#7-database-recovery).

---

## 3. Season end

Reverse the flips to stop wasting quota:

1. Remove the league from `ODDS_API_SPORTS` and `LEAGUES_ENABLED`, then `task down && task up`.
2. Disable its schedule: `bb pipeline schedule set --league NFL --cron "0 12 * * *" --enabled=false`.

Historical data (lines, bets, performance) is unaffected — only ingestion and scheduling stop.

---

## 4. Soccer and hockey specifics

- Soccer (EPL, FIFA_WC) uses **3-way moneylines** — `DRAW` is a valid side on MONEYLINE bets and parlay
  legs ([ADR-027](../decisions/027-three-way-markets-and-regulation-settlement.md)).
- League scope and per-sport data sources are set by
  [ADR-026](../decisions/026-sport-expansion-scope-and-data-sources.md).
- Prop edges only run for leagues in the agent's `PROP_EDGES_LEAGUES` (default FIFA_WC, EPL, MLB) —
  extend it when enabling a prop-capable league.

---

## 5. Quota and key care

- The Odds API free tier is 500 requests/month; each sport key polled costs quota every
  `ODDS_API_POLL_INTERVAL` (default 300 s). More enabled sports = faster burn — widen the poll interval
  in `.env` if needed.
- Remaining quota is reported in Odds API response headers — check
  `task logs:service -- lines-service` or your [the-odds-api.com](https://the-odds-api.com) account page.
- College stats keys (`CFBD_API_KEY`, `CBBD_API_KEY`) are only needed once NCAA_FB / NCAA_BB are enabled.
- Key rotation: update `.env`, then `task down && task up`.
