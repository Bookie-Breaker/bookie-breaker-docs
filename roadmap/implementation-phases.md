# Implementation Phases

BookieBreaker is built in 8 phases (0-7) using vertical slices. Phase 0 bootstraps consistent tooling and repo
structure. Phases 1-3 deliver a working end-to-end system for one sport (NBA). Phases 4-5 add intelligence and
visualization. Phase 6 expands to all leagues — 10 leagues across 5 sports after soccer and hockey were added to
scope ([ADR-026](../decisions/026-sport-expansion-scope-and-data-sources.md)). Phase 7 adds advanced bet types.

NBA is the first sport because: 82-game regular season provides high sample size, nba_api is a mature and
well-documented Python package, and The Odds API has strong NBA coverage.

## Current Status (as of 2026-07-06)

**Phase 0 is complete. Phases 1, 2, 3, 4, and 5 are code-complete pending end-to-end DoD verification. Phase 6
Waves 0 (foundations), 1 (soccer: FIFA_WC + EPL), and 2 (baseball: MLB live, NCAA_BSB dormant) are
code-complete and merged; the soccer live pass must run before the World Cup ends July 19. Waves 3–5
(football, hockey, NCAA basketball) are being built as one combined wave — all four leagues ship dormant
and enable at their season starts (NFL/NCAA_FB September, NHL October, NCAA_BB November).**

**Phase 7 (Advanced Features) is planned and in build as of 2026-07-06.** Full scope was accepted: player props,
live/in-game betting, parlay correlation, and advanced ML across all services, code-complete with live/real-data
verification deferred to the desktop. The 2026 FIFA World Cup (live through ~July 19) is the near-term test case:
soccer same-game parlays (the soccer plugin already computes the joint goal grid §5 needs) and live match-market
betting are WC-testable now; **soccer player props are pulled into scope** (amending
[ADR-026](../decisions/026-sport-expansion-scope-and-data-sources.md)), while NBA/NFL props ship dormant until their
seasons. The phase is structured as five waves — **Wave 0** foundations
(side/prop schema + parlay data model + statistics-service player-rate/box-score surfaces), **Wave 1** parlay
correlation, **Wave 2** live betting (SharpAPI SSE against a stub), **Wave 3** player props, **Wave 4** advanced ML
(ensembles, A/B challenger serving, automated retraining). New/updated decisions: ADR-007 (SharpAPI SSE now active),
ADR-018 (football play-decomposition props), ADR-026/027 amendments, and new
[ADR-028](../decisions/028-parlay-data-model.md) (parlay data model) and
[ADR-029](../decisions/029-prop-line-representation.md) (prop-line representation); ADRs for
live-ingestion transport, correlated-Kelly/MC-joint threshold, and ensemble/A-B serving land with their waves.
statistics-service is added as a Phase 7 participant (box-score endpoint for prop grading + model training — it was
missing from the original "Services Modified" list below).

Phase 5 (Dashboard) landed on 2026-07-05 across six PRs, one per repo:

- **ui** (PR #6): the entire dashboard — SvelteKit 2/Svelte 5 with adapter-node, Tailwind v4 + Skeleton v4,
  and echarts behind a hand-rolled `Chart.svelte` wrapper (deliberate ADR-010 deviation: `svelte-echarts`
  is unmaintained). All backend access is server-side per **ADR-025** (load functions + an allowlist of
  `/api` proxy routes; API types generated from the committed specs via `pnpm gen:api` per ADR-016, fixing
  the root Taskfile's broken `gen` task). Pages: home (agent dashboard aggregate, alert acknowledgement,
  on-demand pipeline run), edges (URL-param filters, client-side sorting, cursor load-more), edge detail
  (probabilities/CIs, feature-importance chart, line movement, lazy simulation distributions with an
  explicit ~2h-expiry state, analysis markdown, bet-this-edge prefill), slate, lines (best-odds
  highlighting, row-expand movement charts), performance (bankroll/ROI, win-rate/CLV, calibration plot,
  grouped breakdowns), and the bet ledger + placement form (side derived from the selection with manual
  override, one idempotency key per form session). Live updates: a shared Redis subscriber bridged to
  browsers over `GET /api/events` SSE with debounced `invalidate()` refetches (payloads never rendered).
  Chat: the ADR-017 sidebar streaming tokens from the agent's new SSE analysis endpoint with page-context
  injection and JSON fallback. Env-gated OTEL server instrumentation (ADR-012); `GET /healthz`; multi-stage
  Dockerfile (fixes the previously-broken docker CI job). 47 vitest (unit + msw integration) and 16
  Playwright e2e (desktop + tablet) against a canned-envelope stub backend — self-contained in CI, with a
  `BB_E2E_STACK=1`-gated full-stack project. Foundation cleanups: stray npm lockfile and divergent root
  `.prettierrc` removed, svelte parser wired into the canonical prettier config, vitest include widened to
  `tests/integration`.
- **agent** (PR #6): streaming LLM analysis — `POST /analysis/stream` (SSE `chunk`/`done`/`error` grammar
  per **ADR-024**) with `stream()` added to the ADR-011 provider protocol (Anthropic `messages.stream`,
  Ollama NDJSON); cached analyses replay as a single chunk (`meta.cached`), pre-stream failures stay JSON,
  and nothing persists on mid-stream failure. Plus `game_external_id` on the edge detail response so the
  UI can chart lines-service movement. 220 tests (no real LLM calls; respx-canned Anthropic SSE / Ollama
  NDJSON bodies, testcontainers SSE-route integration incl. an in-band mid-stream error).
- **bookie-emulator** (PR #4): `GET /performance/calibration` (reliability-diagram bins; the ECE refactored
  to derive from the same `calibration_bins()` so the math has one source of truth) and the
  `events:bet.graded` publisher (fires after `apply_grade` commits on all three grading paths, API result
  vocabulary, fire-and-forget). 140 tests (+22).
- **infra-ops** (PR #16): `ui` service in Docker Compose (port 3000; server-side env only; `REDIS_URL` for
  the SSE bridge; `node -e fetch` healthcheck since node:slim has no wget/curl; `UI_ORIGIN` override).
- **docs** (this PR): agent-api.md `/analysis/stream` + edge-detail `game_external_id`,
  bookie-emulator-api.md `/performance/calibration`, regenerated `agent.yaml`/`bookie-emulator.yaml`,
  redis-schemas `events:bet.graded`, components/ui.md APIs-Consumed table fixed to canonical
  service-prefixed paths, **ADR-023** (kagent evaluated → defer to Phase 7: k8s-native framework on a
  Compose project, and ADR-011/017 + the Phase 4 mcp-server already cover its value adds), **ADR-024**
  (streaming analysis transport), **ADR-025** (UI server proxy + Redis→SSE bridge), roadmap updates.
- **cli** (follow-up PR): regenerated agent + emulator clients from the Phase 5 specs (additive; no new
  commands — calibration and streaming are dashboard features).

**Remaining for Phase 5 (verification session, stronger machine):** `task up` with the ui service and the
dashboard exercised against the live stack; live updates observed end-to-end (pipeline run →
`edge.detected` → edge appears without refresh; grade → `bet.graded` → ledger/performance refresh);
streaming chat against real Anthropic/Ollama; the `BB_E2E_STACK=1` Playwright project; then the Phase 5 DoD
checkboxes below (the Playwright-covered boxes are CI-verified against the stub backend).

Earlier the same day, Phase 4 (Agent Intelligence & MCP) landed across five PRs, one per repo:

- **agent** (PR #5): LLM provider abstraction per ADR-011 (`src/agent/llm/`: Anthropic SDK + Ollama
  `/api/chat` behind one protocol, config-only switching, quality/cheap model tiers with prompt templates for
  edge breakdowns, game previews, performance reviews, daily summaries, and alert descriptions);
  `POST`/`GET /analysis` persisted to the new `agent.analyses` table with token accounting and Redis reuse of
  cached analyses; enhanced alerting (`edge.detected` gains an NL `description` — cheap tier, capped per run,
  deterministic template fallback — with deliveries persisted to `agent.edge_alerts` and new
  `GET /alerts` + acknowledge endpoints); cron scheduling as a croniter asyncio loop over
  `agent.pipeline_schedules` (**ADR-015 amended**: APScheduler v4 never left alpha) with
  `GET`/`POST /schedule`, timezone-aware misfire-graced fires (`trigger=SCHEDULED`), and the daily summary
  job; debounced cooldown-gated event re-runs (`trigger=EVENT`) from `lines.updated`/`stats.updated` for
  scheduled leagues; transparent retries on idempotent upstream calls; health/dashboard now report
  `next_scheduled_run` and a provider-keyed LLM dependency. Migration 0002 (`analyses`, `edge_alerts`,
  `pipeline_schedules`; `query_log` re-deferred). 198 tests (161 unit + 37 testcontainers incl. an
  Ollama-provider app proving the ADR-011 switch); no real LLM calls in CI.
- **mcp-server** (PR #1): the entire service — FastMCP (standalone package, ADR-022) REST-to-MCP bridge with
  **15 tools** (`get_edges`, `get_edge_detail`, `get_slate`, `get_prediction`, `get_lines`, `get_team_stats`,
  `get_player_stats`, `get_simulation`, `place_bet` with edge-derived bodies + UUIDv5 idempotency,
  `get_bet_history`, `get_performance`, `ask_analyst` with scope-inferred analysis type, `run_pipeline`,
  `get_pipeline_status`, `get_health`) and 3 `bookiebreaker://` resources; markdown-formatted outputs;
  actionable ToolError mapping; stdio + **streamable HTTP** transports (SSE deprecated by the MCP spec —
  ADR-022); Dockerfile. Unit tests over the in-memory FastMCP client with respx-mocked backends plus
  real-transport integration tests (stdio subprocess and streamable HTTP against a live stub backend).
- **cli** (PR #5): `bb ask <question...>` (analysis-type inference from `--edge`/`--game`, Glamour markdown
  rendering, dedicated 120s-timeout client for LLM calls) and `bb pipeline schedule list|set`; regenerated
  agent client from the Phase 4 spec.
- **infra-ops** (PR #15): mcp-server in Docker Compose (port 8007, streamable HTTP); agent LLM env
  passthrough defaulting to local Ollama (`phi3:mini`) with no hard ollama dependency (degraded-mode LLM);
  `ollama-init` one-shot model pull closing the Phase 1 gap.
- **docs** (this PR): regenerated `agent.yaml` (analysis/alerts/schedule paths), agent-api.md flipped to
  Phase 4 implemented (+ alerts endpoints + `timezone` on schedules, resolving the components/agent.md
  conflict in favor of shipping the alert REST surface), ADR-015 amendment, new ADR-022, agent DB schema doc
  (three new tables), redis-schemas (`agent:analysis:{type}:{scope}` key, `description` on `edge.detected`),
  reconciled mcp-server tool list, roadmap updates.

**Remaining for Phase 4 (verification session, stronger machine):** live `bb ask`/analysis against real
Anthropic and/or Ollama (`task db:reset` needed for migration 0002; `ollama-init` pulls phi3:mini on first
`task up`); MCP server exercised from a real client (Claude Desktop/Code via stdio, or
`claude mcp add --transport http bookiebreaker http://localhost:8007/mcp`); scheduled + event-triggered runs
observed against the live stack; then the Phase 4 DoD checkboxes below.

Earlier, Phase 3 (First Interface & Paper Trading) landed on 2026-07-04 across five PRs, one per repo:

- **bookie-emulator** (PR #3): the entire service — bet placement with live odds capture from lines-service
  (pinned book or best line), game-start/bankroll validation, game-id reconciliation (`emu:gamemap:*`), and
  Postgres-backed idempotency; claim-based transactional grading via three paths (`events:game.completed`
  subscriber with capped-backoff reconnect, 30-minute polling fallback, manual grade endpoint with force
  semantics); best-effort CLV against closing lines; live performance analytics (ROI, win rate, streaks, Brier,
  10-bin ECE, breakdowns) with `performance_summaries` created but deliberately unpopulated; derived bankroll with
  per-grade snapshots; Alembic-managed `emulator` schema per the amended DDL; full `/api/v1/emulator` contract;
  OTEL; Dockerfile. 118 tests (82 unit + 36 testcontainers integration).
- **agent** (PR #4): orchestration completed on top of the Phase 2 edges math module — on-demand pipeline runs
  (`POST /pipeline/run`, 202 + per-step status, per-league duplicate-run guard) sequencing reconcile → simulate →
  predict → edge detection → auto-bet with per-game error isolation; edge detection against de-vigged prices
  (two-sided grouping per book/market/line, best price across books, league EV thresholds, fractional Kelly,
  quality-score confidence) persisted to the new `agent` Postgres schema (edges, pipeline_runs — the schemas README
  previously said Redis-only; amended in this phase); auto-bettor with deterministic UUIDv5 idempotency keys and
  exposure scaling; `/edges`, `/slate`, `/dashboard`, `/pipeline/runs/{id}`, health; `edge.detected` +
  `prediction.completed` publishers and a resilient five-channel subscriber (staleness marking + cache busting
  only — event-triggered re-runs and cron scheduling defer to Phase 4). 134 tests incl. a full-pipeline
  integration test; E2E stack test gated behind `BB_E2E_STACK=1`.
- **cli** (PR #4): the `bb` terminal interface (Cobra + Lip Gloss) — `edges`, `slate`, `predict`, `lines
[--movement]`, `bet place|list`, `performance [--breakdown]`, `pipeline run|status`, `health`; generated clients
  for lines-service, agent, and bookie-emulator; config precedence flags > env > `~/.config/bookiebreaker/`
  `config.yaml` > defaults; `--format json` on every command; exit codes 0/1/2/3.
- **infra-ops** (PR #14): `agent` schema + `agent_svc` role in init-db; both services in Docker Compose with
  healthchecks; `db:migrate` extended per ADR-019 ordering (lines → predictions → emulator → agent); seed paper
  bets/grades/bankroll trail (roadmap task 25).
- **docs** (this PR): agent DDL doc, contract amendments (agent slate + pipeline-run status endpoints, emulator
  placement fields, Phase 4 markers), redis-schemas additions, ADR-019 ordering, committed OpenAPI artifacts for
  both new services per ADR-021.

**Remaining for Phase 3 (verification session, stronger machine):** full-stack `task up && task db:migrate &&
task db:seed` (existing volumes need `task db:reset` for the new agent schema/role); the gated E2E test
(`BB_E2E_STACK=1`, agent repo) driving pipeline → edge → auto-bet → grade against the live stack; live checks of
statistics-service's `game.completed` publishing, lines-service closing-line behavior, and reconciler match quality
on real Odds API data; then the Phase 3 DoD checkboxes below. Offseason caveats from Phases 1-2 apply (sparse live
NBA data).

Earlier on 2026-07-04, Phase 2 (Prediction Core) landed across six PRs, one per repo:

- **agent** (PR #3): standalone edge-detection math module (`src/agent/edges/`) per
  [edge-detection.md](../algorithms/edge-detection.md) — odds conversions, de-vig (multiplicative/additive/power),
  EV with per-league minimums, fractional Kelly with exposure scaling, edge quality, CLV, edge decay, and
  `detect_edge()`. Pure stdlib, 71 golden-value tests. Two doc-snippet corrections applied (see the algorithms doc
  notes below). Parlay correlation (doc §5) deferred to Phase 7 with the simulation-output work it needs.
- **simulation-engine** (PR #4): the entire service — sport-agnostic Monte Carlo framework, vectorized
  possession-based NBA plugin (closed-form per-possession PMF; 10k iterations ≈ 50 ms against the 10 s DoD),
  dual-criteria convergence (SE always reported; probability-stability is the practical trigger), seed
  reproducibility, Redis-only persistence per redis-schemas.md plus new `sim:run:*`/`sim:latest:*`/idempotency
  keys, `events:simulation.completed`, full `/api/v1/sim` contract, OTEL, Dockerfile, OpenAPI export.
- **prediction-engine** (PR #4): the entire service — Alembic-managed `predictions` schema (DDL per the schema
  doc), feature engineering honestly limited to Phase 1 data (documented exclusions: travel/altitude/h2h/sharp
  money), one unified NBA XGBoost model with market type as a feature registered as three `model_versions` rows
  sharing one artifact, Platt calibration + split-conformal CIs (ADR-014), season walk-forward training,
  **synthetic-bootstrap model baked into the image** (real NBA training deferred — see below), game-id
  reconciliation to lines-service by team name (cached), full `/api/v1/predict` contract incl. `/edges`,
  `events:prediction.completed`, crash-loop-safe startup.
- **infra-ops** (PR #13): both services in Docker Compose (python-based healthchecks; `prediction-models`
  volume), `db:migrate` extended with the prediction-engine Alembic step (ADR-019 ordering).
- **docs** (this PR): generated OpenAPI artifacts committed per ADR-021, contract amendments, roadmap update.
- **cli** (PR): oapi-codegen-generated Go clients for both Python services with a compile check (task 20).

**Remaining for Phase 2 (verification session, stronger machine):** real NBA training data collection
(`prediction-engine/scripts/collect_nba_data.py`, residential IP; offseason data sparsity noted) and a real-data
`train.py` run — the calibration quality targets (ECE < 0.03, Brier < 0.24) are only meaningful then; live-stack
convergence behavior; plus the deferred Phase 1 E2E items below (Grafana/OTEL verification, live endpoints).

## Phase 1 Status (as of 2026-07-03, evening)

**Phase 1 is code-complete pending end-to-end DoD verification.** The remaining Phase 1 build
work landed on 2026-07-03 across three PRs (lines-service #9, statistics-service #5, infra-ops #12), all green in
CI including testcontainers integration suites:

- **lines-service** (PR #9): all 8 read endpoints with opaque keyset cursor pagination and read-time derived fields
  (`implied_probability`, `side`, opening/closing flags); value-based deduplication; closing-line capture driven by
  a new `lines.games` ingestion-metadata table (migration 004); the ingestion scheduler is now actually wired into
  `main.go` (it previously existed but never started) with a no-API-key guard; testcontainers integration tests.
  The tests caught a real bug: snapshot inserts used `ON CONFLICT ON CONSTRAINT` against a unique _index_, so every
  insert failed against a real database — fixed with column-based conflict inference.
- **statistics-service** (PR #5): the entire service — throttled/circuit-broken stats.nba.com adapter (ADR-020),
  ESPN injuries adapter (ADR-008 fallback; NBA.com has no injury endpoint), Redis-first storage with per-type TTLs
  and stale mirrors, deterministic UUIDv5 ids, derived-stat computation with formula fallback, refresh scheduler,
  scoreboard watcher publishing `game.completed` exactly once, the 11 documented Phase 1 endpoints, Dockerfile, and
  unit + integration suites. Deferred contract endpoints are recorded in the
  [statistics-service plan](service-plans/statistics-service.md).
- **infra-ops** (PR #12): both app services added to Docker Compose with healthchecks and OTLP wiring; migrate/seed
  flow documented (`task up && task db:migrate && task db:seed`).

**Remaining for Phase 1 (next session): end-to-end DoD verification** — run the full stack via `task up`, exercise
the live endpoints (needs an `ODDS_API_KEY`; note stats.nba.com may rate-limit or block datacenter IPs, and it is
the NBA offseason so live lines/schedule data will be sparse), verify OTEL traces/metrics/logs in
Grafana/Tempo/Loki, and check off the remaining boxes below. Two open items spotted during the build: the services
do not yet serve their OpenAPI specs at `/docs` or `/swagger` (specs are hand-authored in this repo per ADR-021),
and the lines-service spec's `format: uuid` on `game_id`/`line_id` conflicts with the TEXT external ids the service
stores (see the [lines-service plan](service-plans/lines-service.md) reconciliation note).

Earlier on 2026-07-03: previously open PRs merged (infra-ops #3–#5, lines-service #3–#5, docs #4), CI repairs
(reusable-workflow permissions, actionlint PATH, govulncheck module path, golangci-lint v2, image tags), dependency
refreshes, ADR renumbering (020/021), `statistics_svc` role + `raw_api_responses` grants, Spectral OpenAPI linting,
observability-stack image/config fixes, and a clean full-stack bring-up with migrations and seed data applied.

---

## Phase 0: Repository Bootstrap & Developer Tooling

**Goal:** Establish consistent repository structure, developer tooling, and quality gates across all 11 repos before any
feature code is written.

### Services Built

- **infra-ops** (extended) -- Shared CI workflow updates, root-level Taskfile bootstrap task
- **All repos** (scaffolded) -- Consistent structure, required files, `.config/` directory, hooks

### Key Tasks (ordered)

1. Create `.config/mise.toml` in each of the 11 repos with language-appropriate tools (Go/Python/Node runtimes,
   lefthook, markdownlint-cli2, yamllint, taplo, commitlint, gitleaks, shellcheck, hadolint, actionlint, go-task)
2. Create `.config/lefthook.yml` in each repo with language-specific hooks: pre-commit (lint + format + security),
   commit-msg (commitlint), pre-push (tests + type-check + audit)
3. Create shared config files in each repo's `.config/`: `commitlint.yml`, `markdownlint.jsonc`, `yamllint.yml`,
   `taplo.toml`; plus language-specific configs (`golangci.yml`, `air.toml` for Go repos)
4. Create `.editorconfig` in each repo (shared formatting rules)
5. Scaffold all 11 repos with required files: `.gitignore`, `LICENSE`, `README.md`, `.env.example`, `Taskfile.yml`,
   `renovate.json`
6. Create GitHub templates in each repo: `.github/CODEOWNERS`, `.github/pull_request_template.md`,
   `.github/ISSUE_TEMPLATE/bug_report.yml`, `.github/ISSUE_TEMPLATE/feature_request.yml`
7. Create `.github/workflows/ci.yml` in each repo calling reusable workflows from infra-ops
8. Create `tests/unit/`, `tests/integration/`, `tests/e2e/` directories in each service repo with placeholder test files
9. Run `mise install && lefthook install` in each repo; verify all hooks fire on a test commit
10. Verify a non-conventional commit message is rejected by commitlint
11. Verify gitleaks blocks a commit containing a test secret pattern
12. Update `clone-all.sh` to run `mise install` and `lefthook install` after cloning each repo
13. Add `bootstrap` task to root `BookieBreaker/Taskfile.yml` that runs `mise install && lefthook install` across all
    repos
14. Write the three new docs in bookie-breaker-docs: `operations/repo-standards.md`, `operations/tool-management.md`,
    `operations/git-hooks.md`
15. Create shared `CLAUDE.md` in bookie-breaker-docs with system-wide context (architecture overview, service map, repo
    layout, conventions, shared commands/workflows). Symlink from `BookieBreaker/CLAUDE.md` →
    `bookie-breaker-docs/CLAUDE.md`. Each service repo gets its own `CLAUDE.md` extending the shared config with
    service-specific context. See [Claude Config](../operations/claude-config.md).

### Dependencies

- None (this precedes all feature work)

### Definition of Done

- [x] `mise install` from any repo installs all required tools at pinned versions
- [x] `lefthook install` succeeds in all 11 repos
- [x] A test commit with a non-conventional message is rejected by commit-msg hook
- [x] A test commit with a Python formatting error is rejected by pre-commit hook
- [x] A test commit with a staged secret pattern is rejected by gitleaks
- [x] `task bootstrap` from `BookieBreaker/` root runs mise install + lefthook install for all repos
- [x] All 11 repos have consistent structure matching `operations/repo-standards.md`
- [x] All file types are covered by at least one linter (Go, Python, TS, YAML, JSON, TOML, Markdown, Shell, Dockerfile,
      GitHub Actions)
- [x] CI reusable workflows align with hook tool versions from mise
- [x] Shared `CLAUDE.md` exists in bookie-breaker-docs and is symlinked from `BookieBreaker/CLAUDE.md`
- [x] Each service repo has a `CLAUDE.md` with service-specific context

### Risk Factors

- **mise plugin availability** -- Some tools may not have first-class mise support. Mitigation: use aqua, npm, or pipx
  backends within mise's `[tools]` section.
- **Lefthook + mise PATH** -- Lefthook may not find mise-installed binaries if the shell PATH is not configured.
  Mitigation: set `LEFTHOOK_CONFIG` and ensure `mise activate` runs before git operations.
- **mypy on staged files** -- mypy requires full import context and cannot lint individual files. Mitigation: run mypy
  on `src/` directory in pre-push (not pre-commit), accept the slightly longer run time.

---

## Phase 1: Infrastructure & Data Foundation

**Goal:** Stand up shared infrastructure and both data-layer services for NBA.

### Services Built

- **infra-ops** (new) -- Docker Compose, Taskfile, Postgres+TimescaleDB, Redis, init scripts, OpenTelemetry stack
  (otel-collector + Prometheus + Tempo + Loki + Grafana), Ollama local LLM container
- **statistics-service** (new) -- Go/Echo REST API with NBA data adapter, OTEL instrumentation, raw API response
  archival
- **lines-service** (new) -- Go/Echo REST API with The Odds API integration, OTEL instrumentation, raw API response
  archival

### Key Tasks (ordered)

1. Create infra-ops repo structure: `docker-compose.yml`, `Taskfile.yml`, `.env.example`
2. Configure Postgres 16 + TimescaleDB container with per-service schemas (`lines`, `predictions`, `emulator`)
3. Configure Redis 7 container
4. Write database init scripts (`init-db/03-create-schemas.sql`, `04-create-roles.sql`)
5. Set up root `Taskfile.yml` at `BookieBreaker/` level with `up`, `down`, `build`, `logs` tasks
6. Scaffold statistics-service Go project (`cmd/server/`, `internal/`, `pkg/`)
7. Implement NBA adapter using nba_api (via embedded Python or HTTP wrapper) -- team stats, player stats, schedules,
   game results, injury reports
8. Implement Redis caching layer with configurable TTL for stats responses
9. Implement derived statistics computation: rolling averages, offensive/defensive ratings, pace, efficiency
10. Build REST API endpoints: `/api/v1/stats/teams`, `/api/v1/stats/players`, `/api/v1/stats/games`,
    `/api/v1/stats/schedule` (league selected via `?league=` query parameter)
11. Add health check endpoint (`/healthz`)
12. Scaffold lines-service Go project (`cmd/server/`, `internal/`, `pkg/`)
13. Implement The Odds API polling adapter for NBA (spreads, totals, moneylines)
14. Implement line normalization (American/decimal/fractional odds to canonical format)
15. Set up TimescaleDB hypertable for line snapshots with timestamps
16. Implement ingestion scheduler (configurable poll interval, default 5 minutes)
17. Build REST API endpoints: `/api/v1/lines/current`, `/api/v1/lines/game/{game_id}/movement`,
    `/api/v1/lines/game/{game_id}/closing`
18. Add health check endpoint
19. Set up OpenAPI spec generation for both Go services (Echo middleware or manual spec)
20. Write integration tests for both services against real Postgres/Redis (testcontainers or Docker)
21. Add Dockerfiles for both services (multi-stage builds)
22. Add services to Docker Compose
23. Write seed data script and fixtures for development
24. Set up OpenTelemetry observability stack in Docker Compose: otel-collector, Prometheus, Tempo, Loki, Grafana with
    provisioned dashboards. All services instrument with OTEL SDK from day one
    ([ADR-012](../decisions/012-observability-otel-first.md))
25. Add Ollama container to Docker Compose with configurable model pull, health check, and optional
    `docker-compose.gpu.yml` overlay for GPU passthrough ([ADR-011](../decisions/011-local-llm-strategy.md))
26. Implement raw API response archival in both data services: store every external API response in a
    `raw_api_responses` TimescaleDB hypertable (or MinIO for large payloads) with source, timestamp, response body, and
    HTTP status. This data accumulates from day one for future LLM fine-tuning and training data
27. Configure OTEL instrumentation for both Go services: traces (HTTP middleware + outbound calls), metrics (via OTLP
    exporter replacing direct Prometheus client), structured logs forwarded to otel-collector

### Dependencies

- **Phase 0 complete:** all repos bootstrapped with consistent structure, tooling, and hooks

### Definition of Done

> Code-complete as of 2026-07-03; unchecked items require a full-stack E2E pass (`task up` with live upstreams),
> tracked as the next Phase 1 work item in the Current Status section above.

- [ ] `task up` starts Postgres, Redis, lines-service, and statistics-service with no errors
- [ ] `GET /api/v1/stats/teams?league=nba` returns current NBA team statistics
- [ ] `GET /api/v1/stats/schedule?league=nba` returns current NBA schedule
- [ ] `GET /api/v1/lines/current?league=nba` returns live NBA lines from The Odds API
- [ ] Line snapshots are persisted in TimescaleDB with timestamps (integration-tested; live run pending)
- [ ] Stats responses are cached in Redis (second request is fast, TTL works)
- [ ] Health checks pass for both services (integration-tested; live run pending)
- [ ] OpenAPI specs are generated and accessible at `/docs` or `/swagger` (specs exist in `api-contracts/openapi/`
      per ADR-021 but the services do not serve them yet — open item)
- [x] Integration tests pass in CI (testcontainers suites in both data services, green on the 2026-07-03 PRs)
- [x] Seed data script populates development database (verified during the 2026-07-03 stack bring-up)
- [ ] OTEL traces visible in Grafana/Tempo for cross-service requests
- [ ] Prometheus metrics populated via otel-collector pipeline
- [ ] Logs queryable in Grafana/Loki from all running services
- [ ] Ollama container starts and serves a test model via `/api/chat`
- [ ] Raw API responses archived in `raw_api_responses` hypertable for both data services (integration-tested;
      live run pending)
- [ ] Grafana System Health dashboard shows service status, request rates, and latency

### Risk Factors

- **nba_api rate limiting** -- The nba_api package hits NBA.com endpoints that can rate-limit aggressively. Mitigation:
  generous Redis TTLs (1-4 hours for season stats), request throttling, graceful degradation on 429s.
- **The Odds API quota** -- Free tier has 500 requests/month. Mitigation: start with low poll frequency (every 15
  minutes), cache aggressively, upgrade plan when needed.
- **Go ↔ Python bridge for statistics-service** -- statistics-service is Go but nba_api is Python. Decision
  ([ADR-020](../decisions/020-statistics-data-bridge.md)): Go calls NBA.com endpoints directly for Phase 1-5; Python
  sidecar added at Phase 6 for NFL/MLB/NCAA Baseball data packages. Risk: NBA.com endpoints are undocumented and can
  break seasonally.

---

## Phase 2: Prediction Core

**Goal:** Build the simulation and prediction engines, produce NBA predictions and identify edges.

### Services Built

- **simulation-engine** (new) -- Python/FastAPI Monte Carlo framework + basketball plugin
- **prediction-engine** (new) -- Python/FastAPI XGBoost model for NBA

### Services Modified

- **agent** (partial) -- Edge detection math only (no orchestration yet)

### Key Tasks (ordered)

1. Scaffold simulation-engine Python project (`src/` layout, `pyproject.toml`, FastAPI app)
2. Implement sport-agnostic Monte Carlo framework: configurable iterations, convergence detection, seed management
3. Implement basketball simulation plugin: possession-based simulation using team pace, offensive/defensive efficiency,
   shooting percentages
4. Implement REST API: `POST /api/v1/simulate` accepts team parameters, returns score distributions, margin
   distributions, total distributions
5. Add Redis caching for simulation results (keyed by matchup + parameter hash)
6. Write unit tests for simulation convergence and distribution properties
7. Add Dockerfile and integrate into Docker Compose
8. Scaffold prediction-engine Python project
9. Implement feature engineering pipeline for NBA: rest days, travel distance, home/away splits, injury impact scores,
   recent form (rolling averages)
10. Pull features from statistics-service and lines-service via REST
11. Train initial XGBoost model on historical NBA data (spread, total, moneyline outcomes)
12. Implement calibration layer: Platt scaling or isotonic regression to produce calibrated probabilities
13. Implement confidence interval estimation
14. Implement REST API: `POST /api/v1/predict` accepts game ID or matchup, returns calibrated probabilities with
    confidence intervals for spread/total/moneyline
15. Implement model versioning: store model artifacts with version metadata
16. Add Dockerfile and integrate into Docker Compose
17. Implement edge detection math as a standalone Python module (can be used by agent later): compare calibrated
    probability vs market-implied probability, compute edge size, filter by threshold
18. Write integration tests: simulation → prediction → edge detection pipeline
19. Generate OpenAPI specs for both services (FastAPI auto-generates)
20. Set up OpenAPI codegen pipeline: generate Go clients from Python service specs for use by CLI and other Go services

### Dependencies

- **Phase 1 complete:** statistics-service must be serving NBA stats, lines-service must be serving NBA lines

### Definition of Done

> Code-complete as of 2026-07-04. Endpoints follow the canonical api-contracts paths
> (`POST /api/v1/sim/simulations`, `POST /api/v1/predict/predictions`), not the older shorthand below.
> The initial model is trained on synthetic bootstrap data; calibration-curve quality against the
> ECE/Brier targets requires the real-data training run in the verification session.

- [x] `POST /api/v1/sim/simulations` with two NBA teams returns score/margin/total distributions (10,000+ iterations)
- [x] Simulation results show reasonable convergence (SE always reported; probability-stability criterion converges
      well before 10k iterations — strict SE < 0.005 is unreachable at 10k for basketball margins, see the
      algorithms doc §1)
- [x] `POST /api/v1/predict/predictions` returns calibrated probabilities for spread, total, and moneyline
- [x] Predictions include confidence intervals (split conformal, 90%)
- [x] Edge detection correctly identifies cases where calibrated probability > implied probability
      (agent `detect_edge()` unit-tested; prediction-engine `/edges` integration-tested end-to-end)
- [x] Model version is tracked and retrievable (`model_versions` registry + `/api/v1/predict/models`)
- [x] Feature engineering pulls live data from statistics-service and lines-service (integration-tested against
      mocked contracts; live run pending verification session)
- [x] OpenAPI specs generated, Go clients compile
- [x] All tests pass (agent 71 unit; simulation-engine 47 unit + 10 integration; prediction-engine 42 unit +
      7 integration — all green in CI)
- [ ] Real-data model calibration meets quality targets (ECE < 0.03, Brier < 0.24) — deferred to the
      verification session with the nba_api collection run

### Risk Factors

- **Training data availability** -- Need historical NBA game data with outcomes for model training. Mitigation: nba_api
  provides historical game logs; may need to build a one-time data collection script.
- **Model calibration quality** -- XGBoost out-of-the-box may not produce well-calibrated probabilities. Mitigation:
  Platt scaling as post-processing, evaluate with calibration curves before proceeding.
- **Simulation accuracy vs speed** -- 10,000 iterations per matchup may be slow for full-slate runs. Mitigation: NumPy
  vectorization, start with simpler simulation model, optimize after validation.

---

## Phase 3: First Interface & Paper Trading

**Goal:** Wire everything together with the agent, add paper trading via bookie-emulator, and build the CLI. A user can
see NBA edges, place paper bets, and track performance from the terminal.

### Services Built

- **bookie-emulator** (new) -- Python/FastAPI paper trading system
- **cli** (new) -- Go/Cobra+Charm terminal interface
- **agent** (complete) -- Python/FastAPI pipeline orchestration

### Key Tasks (ordered)

1. Scaffold bookie-emulator Python project
2. Implement bet placement: accept bet type, game, side, odds, stake; persist to Postgres (`emulator` schema)
3. Implement bet grading: subscribe to `game.completed` Redis events, grade bets as win/loss/push using final scores
   from statistics-service
4. Implement performance analytics: ROI, win rate, CLV (compare placement odds to closing odds from lines-service),
   units won/lost
5. Implement bet ledger: full history with filters (sport, bet type, date range, outcome)
6. Build REST API: `POST /api/v1/bets`, `GET /api/v1/bets`, `GET /api/v1/performance`, `GET /api/v1/bets/{id}`
7. Add Dockerfile and integrate into Docker Compose
8. Scaffold agent Python project
9. Implement pipeline orchestration: scheduled runs that sequence statistics-service → simulation-engine →
   prediction-engine → edge detection → alerting
10. Implement edge detection endpoint: `GET /api/v1/edges` (calls prediction-engine, compares to lines-service, returns
    filtered edges)
11. Implement auto-bet: when edge exceeds threshold, place paper bet via bookie-emulator
12. Implement Redis pub/sub: subscribe to `lines.updated`, `stats.updated`, `game.completed`; publish `edge.detected`,
    `prediction.completed`
13. Build REST API: `GET /api/v1/edges`, `POST /api/v1/pipeline/run`, `GET /api/v1/pipeline/status`, `GET /api/v1/slate`
    (today's games with predictions)
14. Add Dockerfile and integrate into Docker Compose
15. Scaffold CLI Go project with Cobra commands
16. Implement `bb edges` -- list current edges (calls agent `/api/v1/edges`)
17. Implement `bb predict <game>` -- show prediction details (calls agent)
18. Implement `bb bet place` -- place a paper bet (calls bookie-emulator)
19. Implement `bb bet list` -- show bet history (calls bookie-emulator)
20. Implement `bb performance` -- show ROI, win rate, CLV (calls bookie-emulator)
21. Implement `bb lines <game>` -- show current lines (calls lines-service directly)
22. Implement `bb slate` -- show today's games with predictions (calls agent)
23. Style output with Lip Gloss and Glamour for polished terminal tables and formatting
24. Write end-to-end test: full pipeline run → edge detected → paper bet placed → game completes → bet graded
25. Update seed data to include paper bets and performance history

### Dependencies

- **Phase 2 complete:** simulation-engine and prediction-engine must be operational
- **Phase 1 services running:** statistics-service and lines-service

### Definition of Done

> Code-complete as of 2026-07-04 (see Current Status). Endpoints follow the canonical api-contracts paths
> (`/api/v1/agent/...`, `/api/v1/emulator/...`), not the older unprefixed shorthand in the task list above.
> Scheduling shipped as on-demand + event reactions per the Phase 3 minimum; cron schedules are Phase 4. All
> boxes below need the full-stack pass in the verification session.

- [ ] `bb edges` displays NBA edges in a formatted terminal table
- [ ] `bb bet place` successfully places a paper bet with current odds
- [ ] `bb performance` shows ROI, win rate, and CLV metrics
- [ ] `bb slate` shows today's NBA games with predictions
- [ ] Agent runs scheduled pipeline (at minimum: on-demand via `POST /api/v1/pipeline/run`)
- [ ] Bookie-emulator grades bets when games complete (via event or polling)
- [ ] CLV is computed by comparing placement odds to closing odds
- [ ] End-to-end test passes: data → simulation → prediction → edge → bet → grade
- [ ] All services start cleanly with `task up`

### Risk Factors

- **Agent complexity** -- The agent coordinates many services; failure in any one breaks the pipeline. Mitigation:
  implement circuit breakers and clear error reporting from the start.
- **Bet grading timing** -- Games complete at unpredictable times; need reliable detection. Mitigation: both
  event-driven (Redis sub to `game.completed`) and polling fallback (check statistics-service periodically for final
  scores).
- **CLI UX** -- Terminal output must be readable and useful without being overwhelming. Mitigation: start with simple
  table output, iterate on formatting.

---

## Phase 4: Agent Intelligence & MCP

**Goal:** Add LLM-powered analysis to the agent and build the MCP server for IDE integration.

### Services Modified

- **agent** -- Add Anthropic SDK integration, natural language analysis, enhanced alerting

### Services Built

- **mcp-server** (new) -- Python/MCP SDK (FastMCP) server

### Key Tasks (ordered)

> Code-complete as of 2026-07-05 (see Current Status). The analysis endpoint shipped at the canonical
> contract path `POST /api/v1/agent/analysis` (not the older `/api/v1/analyze` shorthand below); the MCP
> tool list shipped as the full 15-tool set reconciled across the service plan and component spec; SSE was
> superseded by streamable HTTP per [ADR-022](../decisions/022-mcp-sdk-and-transport.md).

1. ~~Integrate LLM provider abstraction layer into agent~~ — done: Anthropic API (cloud) and Ollama (local),
   config-only switching (`LLM_PROVIDER`, `LLM_BASE_URL`, `LLM_MODEL`). See
   [ADR-011](../decisions/011-local-llm-strategy.md)
2. ~~Implement prompt templates~~ — done: edge analysis, game previews, performance commentary, daily
   summaries, alert descriptions
3. ~~Implement the analysis endpoint~~ — done as `POST /api/v1/agent/analysis` + `GET /analysis/{id}`
4. ~~Wire CLI `bb ask <question>` command to agent analysis endpoint~~ — done (+ `bb pipeline schedule`)
5. ~~Implement daily summary generation~~ — done (DAILY_SUMMARY analyses on their own cron)
6. ~~Enhance alerting~~ — done: NL descriptions on `events:edge.detected`, `edge_alerts` persistence, and
   `GET /alerts` + acknowledge endpoints
7. ~~Implement configurable pipeline schedules~~ — done: croniter loop over `agent.pipeline_schedules`
   (ADR-015 amended), plus debounced event-triggered re-runs
8. ~~Add retry logic and graceful failure handling to pipeline orchestration~~ — done (idempotent-call
   retries with backoff + jitter)
9. ~~Scaffold mcp-server Python project~~ — done
10. ~~Implement MCP protocol lifecycle~~ — done (FastMCP; initialize/capability negotiation/tool listing
    asserted over real transports)
11. ~~Implement MCP tools~~ — done: the roadmap 8 plus `get_edge_detail`, `get_team_stats`,
    `get_player_stats`, `get_simulation`, `run_pipeline`, `get_pipeline_status`, `get_health` (15 total)
12. ~~Implement MCP resources~~ — done: `bookiebreaker://edges/current`, `bookiebreaker://performance/summary`,
    `bookiebreaker://games/today`
13. ~~Support both stdio and SSE transport modes~~ — done as stdio + **streamable HTTP** (ADR-022)
14. ~~Write integration tests for MCP tool invocations~~ — done (stdio subprocess + streamable HTTP against a
    stub backend)
15. ~~Add Dockerfiles and integrate both into Docker Compose~~ — done
16. Test MCP server with Claude Desktop or Claude Code — **verification session**

### Dependencies

- **Phase 3 complete:** agent orchestration, bookie-emulator, and CLI must be working
- **LLM provider** configured (Anthropic API key for cloud, or Ollama running locally from Phase 1)

### Definition of Done

> Code-complete as of 2026-07-05. CI-verified boxes are checked; the remaining boxes need the live stack
> and/or a real MCP client in the verification session.

- [ ] `bb ask "Why do you like the over in Lakers vs Celtics?"` returns coherent LLM-generated analysis
      (end-to-end plumbing CI-tested with mocked LLMs; coherence needs a live Anthropic/Ollama run)
- [x] Agent generates daily edge summaries automatically (scheduler cron + summary service, CI-tested;
      live cadence observed in the verification session)
- [x] MCP server responds to `tools/list` with all implemented tools (15; asserted over stdio and
      streamable HTTP)
- [x] MCP `get_edges` tool returns current edges when called from an MCP client (in-memory, stdio, and HTTP
      clients in CI)
- [x] MCP `place_bet` tool successfully places a paper bet (idempotency-key behavior asserted)
- [x] stdio and streamable HTTP transports work (SSE superseded per
      [ADR-022](../decisions/022-mcp-sdk-and-transport.md))
- [ ] MCP server tested with at least one real MCP client (Claude Desktop, Claude Code, etc.) —
      **verification session**

### Risk Factors

- **LLM cost** -- Anthropic API calls cost money; daily summaries + user questions can add up. Mitigation: cache
  analysis results, use Haiku for routine summaries and Sonnet for detailed analysis, set daily cost limits.
- **Prompt quality** -- LLM output quality depends heavily on prompt engineering. Mitigation: iterate on prompts with
  real data, maintain a prompt library with version control.
- **MCP SDK maturity** -- The MCP SDK is relatively new. Mitigation: stay close to reference implementations, test with
  multiple clients.

---

## Phase 5: Dashboard

**Goal:** Build the web UI for visualizing edges, predictions, line movement, and paper trading performance.

### Services Built

- **ui** (new) -- SvelteKit + ECharts web dashboard

### Evaluation

- **kagent** -- ~~Evaluate kagent (CNCF Sandbox, v0.7.14+)~~ — evaluated and **deferred to Phase 7**
  ([ADR-023](../decisions/023-kagent-evaluation.md)): it is Kubernetes-native (CRDs + Helm) while
  BookieBreaker runs Docker Compose, and its value adds (provider switching, MCP, chat UI) are already
  covered by ADR-011, the Phase 4 mcp-server, and the ADR-017 sidebar whose page-context injection a
  generic chat UI cannot replicate.

### Key Tasks (ordered)

> Code-complete as of 2026-07-05 (see Current Status). Two backend prerequisites were pulled into this
> phase: the agent's streaming analysis endpoint (task 13's "streaming response display" — ADR-024) and
> the emulator's calibration bins endpoint + `events:bet.graded` publisher (tasks 10 and 14 had no data
> source/signal). The UI reaches all backends through its own server per ADR-025 (the Python services
> expose no CORS, and none was added).

1. Scaffold SvelteKit project with Skeleton UI component library
2. Set up API client layer (generated from OpenAPI specs or hand-written fetch wrappers)
3. Build edges dashboard page: table of current edges with filters (sport, bet type, min edge size), sorting by edge
   size/EV/confidence/game time
4. Build prediction detail view: calibrated probabilities, confidence intervals, feature importance chart
5. Build today's slate page: all games with predictions and edges
6. Build current lines page: odds across sportsbooks for each game
7. Build line movement chart component using ECharts (time-series of line changes from open to current)
8. Build simulation distribution visualization: histograms and probability curves using ECharts
9. Build paper trading performance dashboard: ROI over time chart, win rate chart, CLV chart
10. Build calibration plot: predicted probability vs actual outcome frequency
11. Build bet ledger page: filterable/sortable table of all paper bets
12. Build bet placement form: select game, bet type, side, stake
13. Implement LLM analyst chat interface: text input, streaming response display
14. Set up Redis pub/sub subscription for live updates (new edges, bet grading results)
15. Add responsive layout for different screen sizes
16. Write Playwright end-to-end tests for critical user flows
17. Add Dockerfile (Node.js) and integrate into Docker Compose

### Dependencies

- **Phase 3 complete:** agent, bookie-emulator, lines-service, statistics-service must be running
- **Phase 4 complete:** agent LLM analysis for the chat interface

### Definition of Done

> Code-complete as of 2026-07-05. CI-verified boxes (Playwright against the stub backend) are checked;
> the rest need the live stack in the verification session.

- [x] Edges dashboard displays current NBA edges with working filters and sorting (Playwright-verified
      against stub data; live NBA edges need the running stack)
- [x] Clicking an edge shows prediction detail with probabilities and feature importance
      (Playwright-verified)
- [ ] Line movement chart renders correctly for a selected game (component fixture-tested + rendered on
      the edge detail spec; live lines data pending)
- [x] Paper trading dashboard shows ROI over time, win rate, CLV (+ the calibration plot;
      Playwright-verified)
- [x] Bet ledger displays all paper bets with filters (Playwright-verified)
- [x] User can place a paper bet through the UI (Playwright-verified incl. idempotency-key forwarding)
- [ ] LLM chat interface sends questions and displays responses (streaming plumbing CI-tested with mocked
      LLMs end-to-end; coherence needs a live Anthropic/Ollama run)
- [ ] Live updates work: new edge appears without page refresh (bridge + store CI-tested; end-to-end
      observation needs the live stack — `BB_E2E_STACK=1` Playwright project)
- [x] Dashboard is responsive on desktop and tablet (every spec runs on both viewports)
- [x] Playwright tests pass for edge viewing, bet placement, and performance viewing (16 tests, CI)

### Risk Factors

- **ECharts complexity** -- Line movement and distribution charts require careful data formatting. Mitigation: build
  chart components in isolation with mock data first, then wire to real APIs.
- **Real-time updates** -- Redis pub/sub from browser requires SSE or WebSocket proxy. Mitigation: SvelteKit server-sent
  events endpoint that bridges Redis pub/sub to the browser.
- **Scope creep** -- Dashboard features are highly visible and easy to over-build. Mitigation: stick to the defined
  feature list, defer "nice to have" visualizations.

---

## Phase 6: Sport Expansion

**Goal:** Extend the system from NBA-only to all supported leagues. Scope was expanded during Phase 6 planning
(2026-07-05): soccer (FIFA_WC, EPL) and hockey (NHL, NCAA_HKY) were added, bringing the system to **10 leagues
across 5 sports**. See [ADR-026](../decisions/026-sport-expansion-scope-and-data-sources.md) (scope, data-source
matrix, sidecar elimination) and [ADR-027](../decisions/027-three-way-markets-and-regulation-settlement.md)
(three-way markets, DRAW side, regulation-time settlement).

Planning decisions that supersede the original task list below:

- **Data sources re-evaluated** (ADR-026): the ADR-020 Python sidecar is eliminated — every league has a direct
  REST or static-file source consumable from Go. ESPN's site API is the shared real-time scoreboard path for all
  non-NBA/MLB/NHL leagues; MLB uses the official MLB StatsAPI; NFL season stats come from nflverse static CSVs
  (nfl_data_py is deprecated upstream); NHL uses the official NHL API; NCAA Baseball upgrades to ESPN.
- **Season-driven wave order** replaces the code-reuse order: soccer ships first (World Cup live through ~July 19),
  then baseball (MLB mid-season), football (before September kickoff), hockey (before October), NCAA basketball
  (before November). NCAA_BSB is built dormant in the baseball wave; NCAA_HKY is gated on Odds API line coverage.
- **Soccer is competition-config-driven**: one adapter/plugin/model for the SOCCER sport; FIFA_WC and EPL are
  config entries; future competitions are one-day additions.

### Structure: Wave 0 (foundations) + five sport waves

**Wave 0 — Foundations** (cross-cutting; no new leagues turn on):

1. infra-ops: extend `league_enum`/`sport_enum` (fresh volumes) + idempotent `ALTER TYPE ... ADD VALUE IF NOT
EXISTS` migration script as step 0 of `task db:migrate` (existing volumes upgrade without reset)
2. Three-way market support (ADR-027): DRAW side across lines-service derivation, agent N-way de-vig
   (`devig_many`) + N-sided edge grouping, emulator three-way grading, prediction-engine 3-way moneyline rows +
   nullable `side` column, widened check constraints (Alembic), OpenAPI side enums
3. Regulation-score settlement: optional `regulation_home_score`/`regulation_away_score` on
   `events:game.completed` + the game resource; league-aware settlement-score selection in the emulator
4. statistics-service `StatsProvider` seam: neutral DTO package + provider interface, NBA refactored onto it
   (behavior-identical), multi-league config, generalized ESPN client (parameterized sport/league path)
5. simulation-engine `PluginSpec` registry (per-league plugin + params mapper + grid config), sport-aware cache
   hashing (NBA hash unchanged), per-sport spread/total grid radii + push-aware `(p_win, p_push)` covers on
   integer lines
6. prediction-engine `(sport, market)` model registry, per-sport feature registries / artifact paths / synthetic
   bootstrap generators, `--sport` training parameter
7. Docs (this PR set): ADR-026/027, ADR-008/020 amendments, domain models, redis-schemas, OpenAPI specs, enum
   mirror checklist; then regenerated CLI/UI clients

**Wave 1 — Soccer (FIFA_WC + EPL)** [time-critical: WC knockouts end ~July 19]:

1. statistics-service soccer adapter (ESPN, per-competition codes `fifa.world`/`eng.1`): teams, scoreboard,
   standings; regulation score from period linescores; group→REGULAR / knockout→POSTSEASON season mapping
2. simulation-engine soccer plugin: Dixon-Coles-adjusted Poisson goals grid (13×13 PMF, vectorized categorical
   sampling); draws are valid outcomes; output is the regulation score; neutral-site WC vs EPL home advantage
3. prediction-engine: SOCCER feature registry (incl. `sim_draw_probability`, `selection_is_draw`, competition
   one-hot), soccer synthetic bootstrap with DRAW rows, `scripts/collect_soccer_data.py` (pooled ESPN match
   history across competitions), reconciler hardening (NFKD diacritic fold + per-league team-name alias table)
4. agent: FIFA_WC/EPL EV thresholds + market-efficiency entries, schedule seeds (auto_bet=false); infra-ops:
   soccer Odds API keys + DRAW fixture rows; CLI/UI: DRAW rendering, league lists

**Wave 2 — Baseball (MLB live; NCAA_BSB dormant)**:

1. statistics-service MLB adapter (MLB StatsAPI: schedule + probable pitchers, live scores, team/pitcher stats;
   FIP/wOBA computed from counting stats) and NCAA_BSB adapter (ESPN college-baseball); ESPN injuries
   generalization exercised for MLB
2. simulation-engine baseball plugin: calibrated per-half-inning runs distribution (zero-modified geometric),
   9 innings vectorized, starter multiplier (innings 1–6, FIP-derived) + bullpen ERA (7–9), bottom-9 skip, extra
   innings (no draws); NCAA_BSB as config entry
3. prediction-engine MLB features (starter-centric) + baseball synthetic bootstrap + collect script; agent MLB
   schedule seed; infra/CLI/UI/docs sweep

**Wave 3 — Football (NFL + NCAA_FB)** [land before September]:

1. statistics-service NFL adapter (ESPN real-time + nflverse static team-stats CSVs) and NCAA_FB CFBD adapter
   (season stats/SP+ only — never scores; ESPN scoreboard is the watcher)
2. simulation-engine football plugin (drive-based per ADR-018): per-drive {0,3,7} categorical calibrated to
   points-per-drive; key numbers 3/7 emerge naturally; NFL OT allows rare ties (2-way moneyline tie → PUSH);
   NCAA OT alternating possessions until decided
3. prediction-engine NFL (EPA-based) + NCAA_FB (SP+) features, synthetic bootstraps, collect scripts; agent
   schedule seeds; `CFBD_API_KEY`; infra/CLI/UI/docs sweep

**Wave 4 — Hockey (NHL; NCAA_HKY gated)** [land before October]:

1. statistics-service NHL adapter (official NHL API); NCAA_HKY only if Odds API line coverage is confirmed
2. simulation-engine hockey plugin: Poisson goals grid (soccer-plugin reuse) + OT/shootout resolution; NHL
   moneyline/totals settle on final incl. OT/SO; regulation three-way markets deferred
3. prediction-engine NHL features + bootstrap + collect script; agent schedule seed; infra/CLI/UI/docs sweep

**Wave 5 — NCAA Basketball** [land before November; smallest wave]:

1. statistics-service CBBD adapter (season stats/adjusted efficiencies; ESPN scoreboard watcher); simulation
   config-only basketball entry (40-min, college constants); NCAA_BB features/bootstrap/collect; `CBBD_API_KEY`;
   agent schedule seed; infra/CLI/UI/docs sweep

### Dependencies

- **Phases 1–5 code-complete** (they are; live DoD verification proceeds in parallel on the desktop)
- Wave 0 precedes all sport waves; sport waves are then independent and individually shippable
- CFBD and CBBD API keys must be registered (free tiers) before Waves 3 and 5

### Definition of Done

> Every wave verifies via unit + testcontainers CI. Live checkboxes are per-league and follow each league's
> season (soccer July 2026; MLB July 2026; NFL/NCAA_FB September; NHL October; NCAA_BB November; NCAA_BSB and
> NCAA_HKY at their 2027 seasons).

- [ ] Wave 0: golden/parity gates pass (agent de-vig goldens, sim hash parity, statistics-service NBA
      behavior-identical refactor); enum migration applies idempotently to an existing volume
- [ ] All enabled leagues return data from statistics-service
- [ ] All enabled leagues have lines ingested and served by lines-service (incl. DRAW sides for soccer)
- [ ] Simulation-engine has working plugins for basketball, soccer, baseball, football, and hockey
- [ ] Prediction-engine has synthetic-bootstrap models for every sport and collect scripts for every league;
      real-data training per league in its verification session
- [ ] Edge detection works across all leagues, including three-way soccer moneylines
- [ ] Paper trading grades correctly per league (soccer on regulation score incl. DRAW; NFL rare tie → PUSH)
- [ ] CLI and UI support league filtering and display all leagues (incl. DRAW rendering)
- [ ] Integration tests pass for each league's pipeline
- [ ] Soccer live pass during the World Cup window (before July 19): lines→edges→paper bet→grade end-to-end

### Risk Factors

- **ESPN API is undocumented** and now backs six leagues' real-time paths. Mitigation: golden-fixture normalizer
  tests, raw-response archival for drift detection, per-league fallbacks documented in component specs.
- **NCAA data quality** -- CFBD is most reliable; CBBD good; NCAA Baseball/Hockey genuinely weak. Mitigation:
  budget-aware usage (stats only, ESPN for scores), accept lower accuracy, NCAA_HKY gated on line coverage.
- **Soccer team-name reconciliation** (Odds API vs ESPN naming, international diacritics) is the most likely live
  failure. Mitigation: alias table + unmatched-name logging for fast patching during the WC window.
- **Model quality per sport** -- sport-specific feature engineering and synthetic bootstraps per sport; real-data
  calibration validated per league in verification sessions.
- **Scope** -- 9 new leagues across 4 new sports. Mitigation: Wave 0 makes each league wave a self-contained,
  independently shippable unit; season ordering ensures each wave is live-testable soon after it ships.

---

## Phase 7: Advanced Features

**Goal:** Add advanced bet types and modeling capabilities beyond core spread/total/moneyline.

### Services Modified

- **lines-service** -- Live SSE integration (SharpAPI), player prop lines, parlay odds
- **simulation-engine** -- Player stat distributions, in-game state updates, correlation modeling
- **prediction-engine** -- Ensemble methods, neural nets, A/B testing framework
- **agent** -- Live edge detection, prop analysis, parlay recommendations
- **bookie-emulator** -- Live bet tracking, prop grading, parlay grading
- **ui** -- Live dashboard, prop views, parlay builder

### Key Tasks (ordered)

**Player Prop Modeling:**

1. Extend simulation-engine to produce individual player stat distributions from game simulations
2. Implement player prop edge detection for all supported prop types (points, rebounds, assists, yards, etc.)
3. Add player prop lines ingestion to lines-service
4. Add prop views to CLI and UI
5. Add prop bet grading to bookie-emulator

**Live/In-Game Betting:** 6. Integrate SharpAPI SSE stream in lines-service for real-time line updates 7. Implement live
simulation updates in simulation-engine: accept current game state, produce updated distributions 8. Implement live edge
detection in agent with time-sensitivity alerts 9. Build live dashboard in UI with auto-updating odds and edges 10. Add
live bet tracking and grading to bookie-emulator

**Parlay Correlation Analysis:** 11. Implement correlation modeling in simulation-engine: measure dependence between
legs from the same game 12. Implement true parlay probability calculation in prediction-engine (accounting for
correlation) 13. Detect mispriced correlated parlays (same-game parlays where sportsbooks assume independence) 14. Add
parlay builder to UI

**Advanced ML:** 15. Implement ensemble methods in prediction-engine: combine XGBoost with random forests and/or neural
nets 16. Build model A/B testing framework: run multiple model versions in parallel, compare performance 17. Implement
automated model retraining pipeline

### Dependencies

- **Phase 6 complete:** all leagues operational (props and live betting benefit from broad sport coverage)
- SharpAPI subscription for live data

### Definition of Done

- [ ] Player prop edges are detected and displayed for at least NBA and NFL
- [ ] Live lines update in real-time via SSE
- [ ] Live simulation updates produce new edges during games
- [ ] Parlay correlation analysis identifies mispriced same-game parlays
- [ ] Model A/B testing shows comparative performance metrics
- [ ] All new bet types can be paper traded and graded
- [ ] UI supports live updates, prop views, and parlay builder

### Risk Factors

- **Live data latency** -- In-game betting requires sub-second line updates; SSE reliability matters. Mitigation:
  implement reconnection logic, stale-data detection, and fallback polling.
- **Correlation modeling complexity** -- Accurately modeling dependence between parlay legs is statistically
  challenging. Mitigation: start with empirical correlation from simulation output (run simulation once, measure joint
  distributions directly).
- **Model overfitting** -- More complex models (neural nets) risk overfitting on limited sports data. Mitigation: strict
  train/test splits by season, cross-validation, monitor out-of-sample performance.

---

## Phase Summary

| Phase | Name                             | Services                                                         | Est. Effort | Cumulative Value                                                                         |
| ----- | -------------------------------- | ---------------------------------------------------------------- | ----------- | ---------------------------------------------------------------------------------------- |
| 0     | Repository Bootstrap & Tooling   | All repos (scaffolded), infra-ops (extended), shared CLAUDE.md   | M           | Consistent structure, quality gates, and AI-assisted dev config                          |
| 1     | Infrastructure & Data Foundation | infra-ops, statistics-service, lines-service, OTEL stack, Ollama | XL          | Can fetch and store NBA data, observability from day one, LLM training data accumulating |
| 2     | Prediction Core                  | simulation-engine, prediction-engine, agent (partial)            | XL          | Can generate NBA predictions                                                             |
| 3     | First Interface & Paper Trading  | bookie-emulator, cli, agent (complete)                           | L           | Full NBA workflow from terminal                                                          |
| 4     | Agent Intelligence & MCP         | agent (enhanced), mcp-server                                     | M           | LLM analysis, IDE integration                                                            |
| 5     | Dashboard                        | ui                                                               | L           | Visual dashboard operational                                                             |
| 6     | Sport Expansion                  | All services modified                                            | XL          | All 10 leagues / 5 sports supported                                                      |
| 7     | Advanced Features                | All services modified                                            | XL          | Full bet type coverage                                                                   |
