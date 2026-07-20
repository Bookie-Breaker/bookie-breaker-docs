# Playbook 09 — Maintenance

Periodic care: keeping repos and tools current, knowledge graphs fresh, the database tidy, models
retrained, and generated API clients in sync.

---

## 1. Updating the workspace

```bash
./refresh-all-repos.sh    # every repo: stash → default branch → pull --ff-only → restore
task build                # rebuild images so containers run the pulled code
task db:migrate           # apply any new migrations
```

`refresh-all-repos.sh` is safe by design: it auto-stashes dirty trees, only fast-forwards, offers to
delete `[gone]` branches, and prints a problem summary. Related scripts: `checkout-main-all.sh` (switch
all repos to main, skips dirty), `git-status-all.sh` (cross-repo status + CI dashboard).

---

## 2. Toolchains and dependencies

- Tool versions are pinned per repo in `.config/mise.toml`; after pulling, `task bootstrap` re-installs
  toolchains and hooks across all repos. Background:
  [operations/tool-management.md](../operations/tool-management.md).
- Dependency updates arrive as Renovate PRs per repo; CI must pass before merge. **Nothing automerges** —
  branch protection requires a review Renovate cannot give itself, so every PR is merged by hand. Each repo has
  a pinned **Dependency Dashboard** issue listing everything Renovate knows about; that is the place to work
  from, not the PR list.
  - PRs open in a weekly window (before 8am Monday, America/Chicago), capped at 5 concurrent per repo.
  - **Major updates do not open PRs.** They wait as unticked checkboxes on the dashboard; tick one to request
    the PR.
  - Releases must be 5 days old before a PR appears, so a fresh upstream release will not show up immediately.
  - **Security updates ignore all of the above** — they arrive at any hour, skip the age gate, and are labelled
    `security`.
  - Background: [operations/ci-cd-github.md](../operations/ci-cd-github.md) and
    [ADR-033](../decisions/033-dependency-update-strategy.md).

---

## 3. Knowledge graphs

Graphs update automatically via a lefthook post-commit hook (queued, debounced, AST-only). Manual
control from the workspace root:

```bash
task graph:status    # what's queued / stale
task graph:update    # force the debounced runner now
task graph:build     # full rebuild: extract every repo + merge
task graph:merge     # re-merge per-repo graphs into the root graph
```

Never run `graphify update .` at the workspace root — it would replace the merged graph with a
monolithic rescan.

---

## 4. Database

- `task db:migrate` after pulling service changes; if a migration trips on a missing enum value, run
  `bookie-breaker-infra-ops/scripts/migrate-enums.sh` first (idempotent).
- `task db:seed` restores fixture data at any time (idempotent).
- `task db:reset` for a clean slate — destructive; read
  [07 §7](07-troubleshooting.md#7-database-recovery) first, especially the workspace-root vs infra-ops
  distinction.
- Postgres housekeeping (hypertable compression, 12-month raw-response retention) is automatic via
  TimescaleDB policies.

---

## 5. Model lifecycle (retrain and promote)

Retraining is **operator-triggered** via the prediction-engine API ([ADR-032](../decisions/032-ensemble-and-challenger-serving.md));
nothing retrains automatically.

```bash
# 1. Kick off a retrain (per sport; ensemble=true trains the GBT+RF ensemble)
curl -sX POST localhost:8004/api/v1/predict/models/retrain \
  -H 'content-type: application/json' -d '{"sport": "BASKETBALL", "ensemble": true}'
# → {"job_id": ...}; player_prop markets are not retrainable yet (422)

# 2. Poll the job: queued → assembling → training → registered
curl -s localhost:8004/api/v1/predict/models/retrain/<job_id>

# 3. The result registers as a CHALLENGER and shadow-scores on live requests (no bankroll impact).
#    Compare it to the champion once graded samples accumulate:
curl -s localhost:8004/api/v1/predict/models/experiments/BASKETBALL/SPREAD
# → metrics, promotion_ready flag, and blockers

# 4. Promote when ready (422 unless criteria met; {"force": true} overrides)
curl -sX POST localhost:8004/api/v1/predict/models/experiments/BASKETBALL/SPREAD/promote
```

Promotion criteria: challenger beats champion on both Brier and log-loss, calibration (ECE) within
tolerance, over ≥ 300 graded shadow pairs. Promotion flips roles atomically across the sport's game
markets and retires the old champion. To route a share of real (paper) traffic to the challenger before
promoting, set `AB_SPLIT_PCT` on the prediction-engine (default 0).

---

## 6. Regenerating API clients

The OpenAPI specs in [api-contracts/openapi/](../api-contracts/openapi/) are the contract source
([ADR-021](../decisions/021-openapi-spec-strategy.md)). After a spec change:

```bash
task gen    # go generate in lines-service, statistics-service, cli; pnpm gen:api in ui
```

Python services regenerate their exported specs via their own `task spec:export`; the CLI alone can be
regenerated with `task generate:clients` inside `bookie-breaker-cli`.
