# Playbook 04 — Parlays, Props, and Live Betting

The Phase 7 markets: player props, correlation-aware parlays, and in-game (live) betting. Builds on the
core workflow in [03](03-finding-and-betting-edges.md).

---

## 1. Player props

```bash
bb props                     # current PLAYER_PROP edges, best first
bb props --league EPL
bb props --lines             # raw prop lines from the lines service, grouped by game → player → stat
```

Props are enabled per league by the agent's `PROP_EDGES_LEAGUES` setting (default `FIFA_WC`, `EPL`,
`MLB`). Players render by name but are keyed by external id slugs (e.g. `erling-haaland`); stat keys are
canonical (e.g. `player_shots_on_target`) per [ADR-029](../decisions/029-prop-line-representation.md).

Manual prop bet:

```bash
bb bet place --game <game_uuid> --market PLAYER_PROP \
  --selection "Erling Haaland o2.5 shots" --side OVER \
  --player erling-haaland --stat player_shots_on_target --prop-type OVER_UNDER \
  --stake 1 --prob 0.61 --edge 5.0
```

`--prop-type` is `OVER_UNDER` or `YES_NO`.

---

## 2. Parlays

Legs are specified as `game_ext:MARKET:SIDE[:line]`, repeated 2–6 times. Team markets only
(`MONEYLINE`, `SPREAD`, `TOTAL` — no props, [ADR-028](../decisions/028-parlay-data-model.md)); sides are
`HOME`, `AWAY`, `DRAW`, `OVER`, `UNDER`.

Evaluate before committing:

```bash
bb parlay evaluate \
  --leg nba-bos-lal-20260721:MONEYLINE:HOME \
  --leg nba-den-okc-20260721:TOTAL:OVER:224.5 \
  --odds 264            # offered SGP price in American odds; omit to use product of leg decimals
```

The evaluation is correlation-aware
([ADR-030](../decisions/030-parlay-joint-probability-and-correlated-kelly.md)): read the
output as joint probability vs the independence product, the resulting edge, and a Kelly stake
suggestion. `--persist` stores the evaluation even below the EV threshold.

Place (evaluates first, then confirms):

```bash
bb parlay place --leg ...:MONEYLINE:HOME --leg ...:TOTAL:OVER:224.5 --stake 1 --odds 264
# prompts "Place this parlay for 1u? [y/N]" — pass --yes to skip
bb parlay show <bet_id>     # legs and per-leg status after placement
```

Parlay placement always generates a fresh idempotency key (no `--idempotency-key` flag, unlike
`bet place`). Grading resolves per leg as games complete. UI equivalent: `/parlay`.

Automatic parlay scanning during pipeline runs exists but ships disabled (`PARLAY_SCAN_ENABLED=false`,
`PARLAY_AUTO_BET=false` on the agent).

---

## 3. Live betting

Live mode consumes an in-game odds feed over SSE ([ADR-007](../decisions/007-lines-data-sources.md),
[ADR-031](../decisions/031-live-ingestion-transport.md)). Two ways to feed it:

**Credential-free (sharp-stub)** — replays canned frames, good for trying the flow:

```bash
# in bookie-breaker-infra-ops/.env:
#   SHARP_API_URL=http://sharp-stub:8010
cd bookie-breaker-infra-ops
docker compose --profile stubs up -d
```

**Real feed** — set `SHARP_API_URL` and `SHARP_API_KEY` in `bookie-breaker-infra-ops/.env`, then
`task down && task up`. An empty `SHARP_API_URL` disables the live consumer entirely.

Watch live lines and edges:

```bash
bb live                # one snapshot: in-game lines grouped by game, plus live edges
bb live --watch 10     # re-poll every 10 s (min 5) until Ctrl-C
```

Live edge detection in the agent is gated by `LIVE_EDGES_ENABLED` (default **off**); without it you'll
see live lines but no live edges. Place a live bet manually with `bb bet place --live ...`.
UI equivalent: `/live`.

---

## 4. MCP coverage

The MCP server does **not** expose dedicated parlay, props, or live tools — its 15 tools cover the core
workflow (`get_edges`, `get_lines`, `place_bet`, `run_pipeline`, …). From Claude, props are only
indirectly reachable (`get_player_stats`, and `place_bet` accepts prop fields); parlay evaluation and
`bb live` have no MCP equivalent — use the CLI or UI.
