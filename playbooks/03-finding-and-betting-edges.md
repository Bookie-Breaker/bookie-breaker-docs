# Playbook 03 — Finding and Betting Edges

The core loop: review the slate, inspect edges, drill into a game, place a paper bet, and review results.
Everything here works from the CLI (`bb`) or the web UI (<http://localhost:3000>); the CLI commands are
shown with their most useful flags. Prerequisite: a running stack ([02](02-daily-operations.md)) with
edges produced by a pipeline run ([05](05-pipeline-and-scheduling.md)).

---

## 1. The workflow at a glance

```text
pipeline run → edges detected → (auto-bet or manual bet) → games finish → bets graded → performance
```

The agent's pipeline produces predictions, compares them to de-vigged market prices, and stores anything
above the edge threshold. Scheduled runs place paper bets automatically by default
([05 §3](05-pipeline-and-scheduling.md#3-schedules)); this playbook covers the manual path.

---

## 2. Review the slate

```bash
bb slate                       # today's games, predictions, best edge per game
bb slate --date 2026-07-21     # a specific date
bb slate --league NBA
```

UI equivalent: `/slate`.

---

## 3. Inspect edges

```bash
bb edges                             # all current edges, best first
bb edges --market SPREAD             # SPREAD, TOTAL, MONEYLINE
bb edges --min-edge 4 --limit 10
bb edges --date 2026-07-21 --stale   # include stale edges
```

Edges go stale when their underlying line moves or the game completes; stale edges are hidden by default.
UI: `/edges` and `/edges/[id]` for full detail (probability, fair vs offered price, Kelly sizing).

---

## 4. Drill into a game

```bash
bb predict <game_id>                       # calibrated probabilities + top features per market
bb predict <game_id> --market SPREAD,TOTAL
bb lines <game_id>                         # current lines across books (game_id is a UUID)
bb lines <game_id> --movement              # movement history; defaults to SPREAD
bb lines <game_id> --movement --market TOTAL --book draftkings
```

`bb ask` gives an LLM read on anything (see §7). Game UUIDs come from `bb slate --format json` or the UI.

---

## 5. Place a paper bet

```bash
bb bet place \
  --game <game_uuid> \
  --market SPREAD --selection "Boston Celtics" --side HOME \
  --stake 1.5 --prob 0.58 --edge 4.2 \
  --book draftkings --reason "value vs sharp consensus"
```

- Required: `--game`, `--market`, `--selection`, `--side`, `--stake` (units), `--prob` (0–1), `--edge`
  (percentage points).
- Useful optional flags: `--edge-id`/`--prediction-id` (link the bet to its edge), `--kelly`, `--live`,
  and for props `--player`, `--stat`, `--prop-type` ([04 §1](04-parlays-props-and-live.md#1-player-props)).
- Retries: pass `--idempotency-key <key>` to make a retry safe — the emulator dedupes on it. Without it,
  each invocation generates a fresh key and places a new bet.

The emulator captures the odds at placement time; grading happens automatically when the game completes
(`events:game.completed`).

---

## 6. Review results

```bash
bb bet list                                  # recent bets
bb bet list --status open                    # open | graded | all
bb bet list --result WIN --market SPREAD
bb bet list --from 2026-07-01 --to 2026-07-19
bb performance                               # bankroll, ROI, win rate, CLV
bb performance --breakdown league            # league | market_type | sportsbook | month
```

UI: `/bets` for the ledger, `/performance` for charts.

---

## 7. Ask the analyst

```bash
bb ask "which of today's edges has the strongest case?"
bb ask --edge <edge_uuid> "what's driving this edge?"      # edge breakdown
bb ask --game <game_uuid> "preview this game"              # game preview
```

Answers render as Markdown and can take a while (default timeout 120 s; `analysis_timeout` in the CLI
config). Scope resolution: `--type` override, else `--edge` → edge breakdown, else `--game` → game
preview, else performance review.

---

## 8. JSON scripting recipes

Every command supports `--format json`, emitting the unwrapped data object — pipe to `jq`:

```bash
bb edges --format json | jq -r '.[] | "\(.edge_percentage)\t\(.market_type)\t\(.game_id)"' | sort -rn
bb slate --format json | jq -r '.games[].game_id'
bb bet list --status open --format json | jq 'length'
```

Exit codes for scripting: `0` success, `1` API error (incl. unhealthy services), `2` usage error,
`3` connection/timeout. Full CLI reference:
[bookie-breaker-cli README](https://github.com/Bookie-Breaker/bookie-breaker-cli#readme).
