# Playbook 05 — Pipeline and Scheduling

The agent's prediction pipeline is what turns stats and lines into edges and paper bets. This playbook
covers manual runs, cron schedules, event-triggered re-runs, and the knobs that shape a run.

---

## 1. What a pipeline run does

Stages, in order (per league, per game): **simulation → prediction → edge_detection → bet_placement**.

1. Reconcile the league's games, reuse or run Monte Carlo simulations (simulation-engine).
2. Calibrate probabilities per market (prediction-engine), fetch current lines (lines-service).
3. Detect edges against de-vigged prices; size stakes with fractional Kelly under exposure caps.
4. Optionally place paper bets (auto-bet), persist edges, dispatch alerts, publish
   `events:prediction.completed`.

Per-game failures are recorded on the run but never abort it — terminal states are `COMPLETED`,
`COMPLETED_WITH_ERRORS`, or `FAILED`. Architecture detail: [components/agent.md](../components/agent.md).

---

## 2. Manual runs

```bash
bb pipeline run --league NBA                     # returns a run id immediately (async)
bb pipeline run --league NBA --force-refresh     # bypass caches, refetch inputs
bb pipeline run --league NBA --auto-bet=false    # detect edges but place no bets
bb pipeline run --games <uuid>,<uuid>            # restrict to specific games
bb pipeline status <run_id>                      # stages, per-game outcomes, counters
```

`--auto-bet` defaults to **true**. One run per league at a time — a second request while one is active
returns HTTP 409. REST equivalents: `POST /api/v1/agent/pipeline/run`,
`GET /api/v1/agent/pipeline/runs/{id}`.

---

## 3. Schedules

One schedule per league, persisted in `agent.pipeline_schedules`, executed by the agent's croniter-based
scheduler ([ADR-015](../decisions/015-pipeline-scheduler.md)):

```bash
bb pipeline schedule list
bb pipeline schedule set --league NBA --cron "0 10,14,18 * * *" --timezone America/New_York
bb pipeline schedule set --league NBA --cron "0 12 * * *" --min-edge 4 --auto-bet=false
bb pipeline schedule set --league NBA --cron "0 12 * * *" --enabled=false   # pause a league
```

- `--cron` is required; `--timezone` is IANA (default UTC); `--min-edge` (default 3.0) filters auto-bets;
  `--enabled` and `--auto-bet` default true.
- There is no delete — disable with `--enabled=false`.
- Misfires (agent down at fire time) are skipped after a 5-minute grace period, not replayed.

---

## 4. Event-triggered re-runs and the daily summary

The agent listens on Redis and reacts without operator involvement:

| Event                   | Reaction                                                                 |
| ----------------------- | ------------------------------------------------------------------------ |
| `events:lines.updated`  | mark affected edges stale, invalidate caches, request a debounced re-run |
| `events:stats.updated`  | request a re-run for affected leagues                                    |
| `events:game.completed` | mark edges stale, invalidate caches (grading is the emulator's job)      |

Re-runs are debounced (120 s) with a per-league cooldown (600 s) and **only fire for leagues with an
enabled schedule**; the feature is gated by `EVENT_RERUNS_ENABLED` (default on). Live line events route
to the separate live-edge path ([04 §3](04-parlays-props-and-live.md#3-live-betting)).

A daily LLM summary (active edges + performance snapshot) is written to the agent's analyses at 12:00 UTC
by default (`DAILY_SUMMARY_CRON`); it appears in the UI and via `GET /api/v1/agent/analysis/{id}`.

---

## 5. Tuning knobs

Set on the agent (compose env) unless noted:

| Knob                                        | Default           | Effect                                                     |
| ------------------------------------------- | ----------------- | ---------------------------------------------------------- |
| `min_edge_threshold` (per schedule)         | 3.0               | auto-bet filter, via `bb pipeline schedule set --min-edge` |
| `KELLY_MULTIPLIER`                          | 0.25              | fractional Kelly stake scaling                             |
| `MAX_BET_PCT` / `MAX_TOTAL_EXPOSURE`        | 0.05 / 0.15       | per-bet and total bankroll caps                            |
| `PIPELINE_CONCURRENCY`                      | 4                 | games processed in parallel                                |
| `SIMULATION_ITERATIONS` (simulation-engine) | 10000             | Monte Carlo depth vs speed                                 |
| `PROP_EDGES_LEAGUES`                        | FIFA_WC, EPL, MLB | leagues that run the prop step                             |

Apply env changes with `task down && task up`.

---

## 6. When runs fail

| Symptom                    | First check                                                               |
| -------------------------- | ------------------------------------------------------------------------- |
| run stuck `RUNNING`        | `task logs:service -- agent`; upstream health with `bb health`            |
| `COMPLETED_WITH_ERRORS`    | `bb pipeline status <run_id>` — per-game step errors are listed           |
| HTTP 409 on `pipeline run` | a run for that league is already active; wait or check status             |
| edges but no bets          | `--auto-bet=false`? below `--min-edge`? timing framework deferred the bet |

Deeper triage: [07 — Troubleshooting §4](07-troubleshooting.md#4-pipeline-failures) and
[operations/error-handling.md](../operations/error-handling.md).
