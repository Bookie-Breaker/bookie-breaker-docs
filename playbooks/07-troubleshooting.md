# Playbook 07 — Troubleshooting

Symptom-first triage and recovery. Design background on failure modes lives in
[operations/error-handling.md](../operations/error-handling.md).

---

## 1. Triage flow

1. `bb health` — which of the five core services is unhealthy?
2. `docker compose -f bookie-breaker-infra-ops/docker-compose.yml ps` — container state and health for
   everything else (postgres, redis, ollama, observability).
3. `task logs:service -- <name>` — read the failing service's logs.
4. Still unclear? Grafana (<http://localhost:3001>) for error rates and latency, Tempo for a failing
   request's trace ([08](08-observability.md)).

`bb` exit codes during triage: `1` = a service answered with an error, `3` = connection refused/timeout
(service or stack down).

---

## 2. Stack won't start

| Symptom                               | Fix                                                                                                                                          |
| ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `port is already allocated`           | something on the host holds a port from the [port table](02-daily-operations.md#2-service-and-port-reference); `lsof -i :<port>` and stop it |
| a service restarts in a loop          | `task logs:service -- <name>`; commonly a missing sibling repo (images build from `../bookie-breaker-*`) — run `clone-all.sh`                |
| `db:migrate` precondition fails       | lines-service, prediction-engine, bookie-emulator, and agent repos must be cloned next to infra-ops                                          |
| slow first `task up`                  | expected: image builds + ollama model pull (~2.2 GB); nothing blocks on the pull                                                             |
| containers unhealthy right after `up` | dependencies gate on health — give postgres/redis ~30 s, then re-check `ps`                                                                  |

---

## 3. Empty slate or no data

| Symptom          | Likely cause                                                                                                                                                                                |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bb slate` empty | league not in `LEAGUES_ENABLED`, or statistics-service hasn't synced schedules yet                                                                                                          |
| `bb edges` empty | no pipeline run yet — `bb pipeline run --league <X>` ([05](05-pipeline-and-scheduling.md))                                                                                                  |
| no fresh lines   | `ODDS_API_KEY` unset (ingestion disabled; seeded data only), sport key missing from `ODDS_API_SPORTS`, or monthly quota exhausted ([06 §5](06-seasonal-operations.md#5-quota-and-key-care)) |
| props empty      | league not in the agent's `PROP_EDGES_LEAGUES`                                                                                                                                              |

---

## 4. Pipeline failures

- `bb pipeline status <run_id>` lists per-game, per-stage errors — a run continues past individual game
  failures (`COMPLETED_WITH_ERRORS`).
- Stage failing consistently → check that stage's service: simulation (needs statistics-service),
  prediction (needs models present in the `prediction-models` volume), bet placement (emulator).
- HTTP 409 on run: one active run per league; wait for it or check its status.
- Scheduled runs not firing: `bb pipeline schedule list` — enabled? next-run sane? Note misfires while
  the agent is down are skipped after a 5-minute grace, not replayed.
- Recovery patterns (circuit breakers, retries, data-quality gates):
  [operations/error-handling.md](../operations/error-handling.md).

---

## 5. LLM problems

| Symptom                     | Fix                                                                                                                                       |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `bb ask` times out          | first call after start may wait on the Ollama model pull; check `task logs:service -- ollama-init`. Timeout is 120 s (`analysis_timeout`) |
| `bb ask` errors immediately | agent LLM config: `LLM_PROVIDER` (`ollama` default \| `anthropic`), `ANTHROPIC_API_KEY` set if anthropic                                  |
| answers are weak            | `phi3:mini` is the small default — set `OLLAMA_MODEL` to a larger model or switch provider, then `task down && task up`                   |
| model pull failed           | `docker compose -f bookie-breaker-infra-ops/docker-compose.yml run --rm ollama-init` to re-pull                                           |

---

## 6. Live and SSE problems

| Symptom                      | Fix                                                                                                                                    |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `bb live` shows nothing      | live consumer disabled (`SHARP_API_URL` empty) — configure it or start the stub ([04 §3](04-parlays-props-and-live.md#3-live-betting)) |
| live lines but no live edges | `LIVE_EDGES_ENABLED` defaults off on the agent                                                                                         |
| UI doesn't update live       | UI SSE degrades to heartbeats without `REDIS_URL`; check redis health and the browser's `/api/events` stream                           |

---

## 7. Database recovery

`task db:reset` from the workspace root: stops the stack, **deletes the Postgres data volume**
(`bookie-breaker-infra-ops_postgres-data`), restarts, migrates, and seeds. All lines, bets, edges, and
performance history are lost — models, Ollama, and metrics volumes survive.

> **Warning:** running `task db:reset` from inside `bookie-breaker-infra-ops/` instead uses
> `docker compose down -v`, which deletes **every** volume — including trained models, the pulled Ollama
> model, and all metrics/traces/logs. Prefer the workspace-root task.

Targeted fixes short of a reset:

- Enum errors after a service update (`invalid input value for enum`): run
  `bookie-breaker-infra-ops/scripts/migrate-enums.sh` (idempotent), which the root `db:migrate` does not
  include.
- Password confusion: `POSTGRES_PASSWORD` rotates only the `bookiebreaker` superuser; the per-service
  roles (`lines_svc`, `agent_svc`, …) are hard-coded to `localdev` in `init-db/04-create-roles.sql`.
- Re-seed fixtures only: `task db:seed` (idempotent).

---

## 8. Commits blocked by hooks or lint

Lefthook runs lint/format/commit-message checks in every repo. When a commit is rejected: fix the
reported problem — never bypass with `LEFTHOOK=0` or `--no-verify`. Conventional-commit format is
enforced (`type(scope): description`). Hook reference: [operations/git-hooks.md](../operations/git-hooks.md);
style rules: [operations/repo-standards.md](../operations/repo-standards.md).
