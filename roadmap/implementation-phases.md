# Implementation Phases

BookieBreaker is built in 8 phases (0-7) using vertical slices. Phase 0 bootstraps consistent tooling and repo structure. Phases 1-3 deliver a working end-to-end system for one sport (NBA). Phases 4-5 add intelligence and visualization. Phase 6 expands to all leagues. Phase 7 adds advanced bet types.

NBA is the first sport because: 82-game regular season provides high sample size, nba_api is a mature and well-documented Python package, and The Odds API has strong NBA coverage.

---

## Phase 0: Repository Bootstrap & Developer Tooling

**Goal:** Establish consistent repository structure, developer tooling, and quality gates across all 11 repos before any feature code is written.

### Services Built
- **infra-ops** (extended) -- Shared CI workflow updates, root-level Taskfile bootstrap task
- **All repos** (scaffolded) -- Consistent structure, required files, `.config/` directory, hooks

### Key Tasks (ordered)

1. Create `.config/mise.toml` in each of the 11 repos with language-appropriate tools (Go/Python/Node runtimes, lefthook, markdownlint-cli2, yamllint, taplo, commitlint, gitleaks, shellcheck, hadolint, actionlint, go-task)
2. Create `.config/lefthook.yml` in each repo with language-specific hooks: pre-commit (lint + format + security), commit-msg (commitlint), pre-push (tests + type-check + audit)
3. Create shared config files in each repo's `.config/`: `commitlint.yml`, `markdownlint.jsonc`, `yamllint.yml`, `taplo.toml`; plus language-specific configs (`golangci.yml`, `air.toml` for Go repos)
4. Create `.editorconfig` in each repo (shared formatting rules)
5. Scaffold all 11 repos with required files: `.gitignore`, `LICENSE`, `README.md`, `.env.example`, `Taskfile.yml`, `renovate.json`
6. Create GitHub templates in each repo: `.github/CODEOWNERS`, `.github/pull_request_template.md`, `.github/ISSUE_TEMPLATE/bug_report.yml`, `.github/ISSUE_TEMPLATE/feature_request.yml`
7. Create `.github/workflows/ci.yml` in each repo calling reusable workflows from infra-ops
8. Create `tests/unit/`, `tests/integration/`, `tests/e2e/` directories in each service repo with placeholder test files
9. Run `mise install && lefthook install` in each repo; verify all hooks fire on a test commit
10. Verify a non-conventional commit message is rejected by commitlint
11. Verify gitleaks blocks a commit containing a test secret pattern
12. Update `clone-all.sh` to run `mise install` and `lefthook install` after cloning each repo
13. Add `bootstrap` task to root `BookieBreaker/Taskfile.yml` that runs `mise install && lefthook install` across all repos
14. Write the three new docs in bookie-breaker-docs: `operations/repo-standards.md`, `operations/tool-management.md`, `operations/git-hooks.md`

### Dependencies
- None (this precedes all feature work)

### Definition of Done
- [ ] `mise install` from any repo installs all required tools at pinned versions
- [ ] `lefthook install` succeeds in all 11 repos
- [ ] A test commit with a non-conventional message is rejected by commit-msg hook
- [ ] A test commit with a Python formatting error is rejected by pre-commit hook
- [ ] A test commit with a staged secret pattern is rejected by gitleaks
- [ ] `task bootstrap` from `BookieBreaker/` root runs mise install + lefthook install for all repos
- [ ] All 11 repos have consistent structure matching `operations/repo-standards.md`
- [ ] All file types are covered by at least one linter (Go, Python, TS, YAML, JSON, TOML, Markdown, Shell, Dockerfile, GitHub Actions)
- [ ] CI reusable workflows align with hook tool versions from mise

### Risk Factors
- **mise plugin availability** -- Some tools may not have first-class mise support. Mitigation: use aqua, npm, or pipx backends within mise's `[tools]` section.
- **Lefthook + mise PATH** -- Lefthook may not find mise-installed binaries if the shell PATH is not configured. Mitigation: set `LEFTHOOK_CONFIG` and ensure `mise activate` runs before git operations.
- **mypy on staged files** -- mypy requires full import context and cannot lint individual files. Mitigation: run mypy on `src/` directory in pre-push (not pre-commit), accept the slightly longer run time.

---

## Phase 1: Infrastructure & Data Foundation

**Goal:** Stand up shared infrastructure and both data-layer services for NBA.

### Services Built
- **infra-ops** (new) -- Docker Compose, Taskfile, Postgres+TimescaleDB, Redis, init scripts
- **statistics-service** (new) -- Go/Echo REST API with NBA data adapter
- **lines-service** (new) -- Go/Echo REST API with The Odds API integration

### Key Tasks (ordered)

1. Create infra-ops repo structure: `docker-compose.yml`, `Taskfile.yml`, `.env.example`
2. Configure Postgres 16 + TimescaleDB container with per-service schemas (`lines`, `predictions`, `emulator`)
3. Configure Redis 7 container
4. Write database init scripts (`init-db/01-create-schemas.sql`, `02-create-roles.sql`)
5. Set up root `Taskfile.yml` at `BookieBreaker/` level with `up`, `down`, `build`, `logs` tasks
6. Scaffold statistics-service Go project (`cmd/server/`, `internal/`, `pkg/`)
7. Implement NBA adapter using nba_api (via embedded Python or HTTP wrapper) -- team stats, player stats, schedules, game results, injury reports
8. Implement Redis caching layer with configurable TTL for stats responses
9. Implement derived statistics computation: rolling averages, offensive/defensive ratings, pace, efficiency
10. Build REST API endpoints: `/api/v1/stats/nba/teams`, `/api/v1/stats/nba/players`, `/api/v1/stats/nba/games`, `/api/v1/stats/nba/schedule`
11. Add health check endpoint (`/healthz`)
12. Scaffold lines-service Go project (`cmd/server/`, `internal/`, `pkg/`)
13. Implement The Odds API polling adapter for NBA (spreads, totals, moneylines)
14. Implement line normalization (American/decimal/fractional odds to canonical format)
15. Set up TimescaleDB hypertable for line snapshots with timestamps
16. Implement ingestion scheduler (configurable poll interval, default 5 minutes)
17. Build REST API endpoints: `/api/v1/lines/current`, `/api/v1/lines/history`, `/api/v1/lines/closing`
18. Add health check endpoint
19. Set up OpenAPI spec generation for both Go services (Echo middleware or manual spec)
20. Write integration tests for both services against real Postgres/Redis (testcontainers or Docker)
21. Add Dockerfiles for both services (multi-stage builds)
22. Add services to Docker Compose
23. Write seed data script and fixtures for development

### Dependencies
- **Phase 0 complete:** all repos bootstrapped with consistent structure, tooling, and hooks

### Definition of Done
- [ ] `task up` starts Postgres, Redis, lines-service, and statistics-service with no errors
- [ ] `GET /api/v1/stats/nba/teams` returns current NBA team statistics
- [ ] `GET /api/v1/stats/nba/schedule` returns current NBA schedule
- [ ] `GET /api/v1/lines/current?sport=basketball_nba` returns live NBA lines from The Odds API
- [ ] Line snapshots are persisted in TimescaleDB with timestamps
- [ ] Stats responses are cached in Redis (second request is fast, TTL works)
- [ ] Health checks pass for both services
- [ ] OpenAPI specs are generated and accessible at `/docs` or `/swagger`
- [ ] Integration tests pass in CI
- [ ] Seed data script populates development database

### Risk Factors
- **nba_api rate limiting** -- The nba_api package hits NBA.com endpoints that can rate-limit aggressively. Mitigation: generous Redis TTLs (1-4 hours for season stats), request throttling, graceful degradation on 429s.
- **The Odds API quota** -- Free tier has 500 requests/month. Mitigation: start with low poll frequency (every 15 minutes), cache aggressively, upgrade plan when needed.
- **Go ↔ Python bridge for statistics-service** -- statistics-service is Go but nba_api is Python. Mitigation: either use a Python sidecar process that statistics-service calls via HTTP, or use a Go-native HTTP client to hit NBA.com endpoints directly (trading the nba_api convenience for language consistency).

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
3. Implement basketball simulation plugin: possession-based simulation using team pace, offensive/defensive efficiency, shooting percentages
4. Implement REST API: `POST /api/v1/simulate` accepts team parameters, returns score distributions, margin distributions, total distributions
5. Add Redis caching for simulation results (keyed by matchup + parameter hash)
6. Write unit tests for simulation convergence and distribution properties
7. Add Dockerfile and integrate into Docker Compose
8. Scaffold prediction-engine Python project
9. Implement feature engineering pipeline for NBA: rest days, travel distance, home/away splits, injury impact scores, recent form (rolling averages)
10. Pull features from statistics-service and lines-service via REST
11. Train initial XGBoost model on historical NBA data (spread, total, moneyline outcomes)
12. Implement calibration layer: Platt scaling or isotonic regression to produce calibrated probabilities
13. Implement confidence interval estimation
14. Implement REST API: `POST /api/v1/predict` accepts game ID or matchup, returns calibrated probabilities with confidence intervals for spread/total/moneyline
15. Implement model versioning: store model artifacts with version metadata
16. Add Dockerfile and integrate into Docker Compose
17. Implement edge detection math as a standalone Python module (can be used by agent later): compare calibrated probability vs market-implied probability, compute edge size, filter by threshold
18. Write integration tests: simulation → prediction → edge detection pipeline
19. Generate OpenAPI specs for both services (FastAPI auto-generates)
20. Set up OpenAPI codegen pipeline: generate Go clients from Python service specs for use by CLI and other Go services

### Dependencies
- **Phase 1 complete:** statistics-service must be serving NBA stats, lines-service must be serving NBA lines

### Definition of Done
- [ ] `POST /api/v1/simulate` with two NBA teams returns score/margin/total distributions (10,000+ iterations)
- [ ] Simulation results show reasonable convergence (standard error < 1% of mean)
- [ ] `POST /api/v1/predict` returns calibrated probabilities for spread, total, and moneyline
- [ ] Predictions include confidence intervals
- [ ] Edge detection correctly identifies cases where calibrated probability > implied probability
- [ ] Model version is tracked and retrievable
- [ ] Feature engineering pulls live data from statistics-service and lines-service
- [ ] OpenAPI specs generated, Go clients compile
- [ ] All tests pass

### Risk Factors
- **Training data availability** -- Need historical NBA game data with outcomes for model training. Mitigation: nba_api provides historical game logs; may need to build a one-time data collection script.
- **Model calibration quality** -- XGBoost out-of-the-box may not produce well-calibrated probabilities. Mitigation: Platt scaling as post-processing, evaluate with calibration curves before proceeding.
- **Simulation accuracy vs speed** -- 10,000 iterations per matchup may be slow for full-slate runs. Mitigation: NumPy vectorization, start with simpler simulation model, optimize after validation.

---

## Phase 3: First Interface & Paper Trading

**Goal:** Wire everything together with the agent, add paper trading via bookie-emulator, and build the CLI. A user can see NBA edges, place paper bets, and track performance from the terminal.

### Services Built
- **bookie-emulator** (new) -- Python/FastAPI paper trading system
- **cli** (new) -- Go/Cobra+Charm terminal interface
- **agent** (complete) -- Python/FastAPI pipeline orchestration

### Key Tasks (ordered)

1. Scaffold bookie-emulator Python project
2. Implement bet placement: accept bet type, game, side, odds, stake; persist to Postgres (`emulator` schema)
3. Implement bet grading: subscribe to `game.completed` Redis events, grade bets as win/loss/push using final scores from statistics-service
4. Implement performance analytics: ROI, win rate, CLV (compare placement odds to closing odds from lines-service), units won/lost
5. Implement bet ledger: full history with filters (sport, bet type, date range, outcome)
6. Build REST API: `POST /api/v1/bets`, `GET /api/v1/bets`, `GET /api/v1/performance`, `GET /api/v1/bets/{id}`
7. Add Dockerfile and integrate into Docker Compose
8. Scaffold agent Python project
9. Implement pipeline orchestration: scheduled runs that sequence statistics-service → simulation-engine → prediction-engine → edge detection → alerting
10. Implement edge detection endpoint: `GET /api/v1/edges` (calls prediction-engine, compares to lines-service, returns filtered edges)
11. Implement auto-bet: when edge exceeds threshold, place paper bet via bookie-emulator
12. Implement Redis pub/sub: subscribe to `lines.updated`, `stats.updated`, `game.completed`; publish `edge.detected`, `prediction.completed`
13. Build REST API: `GET /api/v1/edges`, `POST /api/v1/pipeline/run`, `GET /api/v1/pipeline/status`, `GET /api/v1/slate` (today's games with predictions)
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
- **Agent complexity** -- The agent coordinates many services; failure in any one breaks the pipeline. Mitigation: implement circuit breakers and clear error reporting from the start.
- **Bet grading timing** -- Games complete at unpredictable times; need reliable detection. Mitigation: both event-driven (Redis sub to `game.completed`) and polling fallback (check statistics-service periodically for final scores).
- **CLI UX** -- Terminal output must be readable and useful without being overwhelming. Mitigation: start with simple table output, iterate on formatting.

---

## Phase 4: Agent Intelligence & MCP

**Goal:** Add LLM-powered analysis to the agent and build the MCP server for IDE integration.

### Services Modified
- **agent** -- Add Anthropic SDK integration, natural language analysis, enhanced alerting

### Services Built
- **mcp-server** (new) -- Python/MCP SDK (FastMCP) server

### Key Tasks (ordered)

1. Integrate Anthropic SDK into agent
2. Implement prompt templates for edge analysis ("Why does this edge exist?"), game previews, and performance commentary
3. Implement `POST /api/v1/analyze` endpoint: accepts a question + context, returns LLM-generated analysis
4. Wire CLI `bb ask <question>` command to agent analysis endpoint
5. Implement daily summary generation: all edges across the slate with commentary
6. Enhance alerting: push edge notifications via Redis pub/sub with natural language descriptions
7. Implement configurable pipeline schedules (e.g., "run 2 hours before first NBA game")
8. Add retry logic and graceful failure handling to pipeline orchestration
9. Scaffold mcp-server Python project
10. Implement MCP protocol lifecycle: initialization, capability negotiation, tool listing
11. Implement MCP tools: `get_edges`, `get_prediction`, `place_bet`, `get_performance`, `get_bet_history`, `get_lines`, `ask_analyst`, `get_slate`
12. Implement MCP resources: current edges summary, recent performance summary
13. Support both stdio and SSE transport modes
14. Write integration tests for MCP tool invocations
15. Add Dockerfiles and integrate both into Docker Compose
16. Test MCP server with Claude Desktop or VS Code extension

### Dependencies
- **Phase 3 complete:** agent orchestration, bookie-emulator, and CLI must be working
- **Anthropic API key** required

### Definition of Done
- [ ] `bb ask "Why do you like the over in Lakers vs Celtics?"` returns coherent LLM-generated analysis
- [ ] Agent generates daily edge summaries automatically
- [ ] MCP server responds to `tools/list` with all implemented tools
- [ ] MCP `get_edges` tool returns current edges when called from an MCP client
- [ ] MCP `place_bet` tool successfully places a paper bet
- [ ] Both stdio and SSE transports work
- [ ] MCP server tested with at least one real MCP client (Claude Desktop, VS Code, etc.)

### Risk Factors
- **LLM cost** -- Anthropic API calls cost money; daily summaries + user questions can add up. Mitigation: cache analysis results, use Haiku for routine summaries and Sonnet for detailed analysis, set daily cost limits.
- **Prompt quality** -- LLM output quality depends heavily on prompt engineering. Mitigation: iterate on prompts with real data, maintain a prompt library with version control.
- **MCP SDK maturity** -- The MCP SDK is relatively new. Mitigation: stay close to reference implementations, test with multiple clients.

---

## Phase 5: Dashboard

**Goal:** Build the web UI for visualizing edges, predictions, line movement, and paper trading performance.

### Services Built
- **ui** (new) -- SvelteKit + ECharts web dashboard

### Key Tasks (ordered)

1. Scaffold SvelteKit project with Skeleton UI component library
2. Set up API client layer (generated from OpenAPI specs or hand-written fetch wrappers)
3. Build edges dashboard page: table of current edges with filters (sport, bet type, min edge size), sorting by edge size/EV/confidence/game time
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
- [ ] Edges dashboard displays current NBA edges with working filters and sorting
- [ ] Clicking an edge shows prediction detail with probabilities and feature importance
- [ ] Line movement chart renders correctly for a selected game
- [ ] Paper trading dashboard shows ROI over time, win rate, CLV
- [ ] Bet ledger displays all paper bets with filters
- [ ] User can place a paper bet through the UI
- [ ] LLM chat interface sends questions and displays responses
- [ ] Live updates work: new edge appears without page refresh
- [ ] Dashboard is responsive on desktop and tablet
- [ ] Playwright tests pass for edge viewing, bet placement, and performance viewing

### Risk Factors
- **ECharts complexity** -- Line movement and distribution charts require careful data formatting. Mitigation: build chart components in isolation with mock data first, then wire to real APIs.
- **Real-time updates** -- Redis pub/sub from browser requires SSE or WebSocket proxy. Mitigation: SvelteKit server-sent events endpoint that bridges Redis pub/sub to the browser.
- **Scope creep** -- Dashboard features are highly visible and easy to over-build. Mitigation: stick to the defined feature list, defer "nice to have" visualizations.

---

## Phase 6: Sport Expansion

**Goal:** Extend the system from NBA-only to all 6 supported leagues.

### Services Modified
- **statistics-service** -- Add adapters for NFL (nfl_data_py), MLB (pybaseball), NCAA (CFBD API, sports-reference)
- **lines-service** -- Add sport-specific line types and markets
- **simulation-engine** -- Add football and baseball simulation plugins
- **prediction-engine** -- Train sport-specific models for each league
- **agent** -- League-specific scheduling, multi-sport edge detection
- **bookie-emulator** -- Multi-sport bet grading
- **cli** -- Sport filter flags for all commands
- **ui** -- Sport/league selector, sport-specific visualizations

### Expansion Order
1. **NFL** -- Most similar pipeline to NBA; drive-based simulation; nfl_data_py is excellent
2. **MLB** -- Plate-appearance simulation; pybaseball is mature; very different sport model
3. **NCAA Basketball** -- Reuses NBA basketball plugin with minor adjustments; CFBD/sports-reference for data
4. **NCAA Football** -- Reuses NFL football plugin; CFBD API for data
5. **NCAA Baseball** -- Reuses MLB plugin; data availability is the weakest of all leagues

### Key Tasks (ordered)

**NFL (first expansion):**
1. Implement NFL adapter in statistics-service (nfl_data_py): team stats, player stats, schedules, results
2. Implement football simulation plugin in simulation-engine: drive-based or play-level simulation
3. Add NFL lines ingestion to lines-service (The Odds API supports NFL)
4. Train NFL XGBoost model in prediction-engine
5. Test full NFL pipeline end-to-end
6. Update CLI and UI with NFL data

**MLB:**
7. Implement MLB adapter in statistics-service (pybaseball): team stats, pitcher stats, batter stats, schedules, results
8. Implement baseball simulation plugin: plate-appearance resolution, pitcher-batter matchups
9. Add MLB lines ingestion
10. Train MLB model
11. Test full MLB pipeline
12. Update CLI and UI

**NCAA Basketball:**
13. Implement NCAA Basketball adapter in statistics-service
14. Configure basketball simulation plugin for college game rules (shot clock, game length)
15. Add NCAA Basketball lines ingestion
16. Train NCAA Basketball model
17. Test and update interfaces

**NCAA Football:**
18. Implement NCAA Football adapter in statistics-service (CFBD API)
19. Configure football simulation plugin for college rules
20. Add NCAA Football lines ingestion
21. Train NCAA Football model
22. Test and update interfaces

**NCAA Baseball:**
23. Implement NCAA Baseball adapter in statistics-service
24. Configure baseball simulation plugin for college rules (aluminum bats, etc.)
25. Add NCAA Baseball lines ingestion
26. Train NCAA Baseball model
27. Test and update interfaces

### Dependencies
- **Phase 3 complete:** full NBA pipeline working end-to-end
- Each sport expansion is independent and can be done in any order, but the recommended order maximizes code reuse

### Definition of Done
- [ ] All 6 leagues return data from statistics-service
- [ ] All 6 leagues have lines ingested and served by lines-service
- [ ] Simulation-engine has working plugins for football, basketball, and baseball
- [ ] Prediction-engine has trained models for all 6 leagues
- [ ] Edge detection works across all leagues
- [ ] Paper trading works for all leagues with correct grading
- [ ] CLI and UI support league filtering and display all leagues
- [ ] Integration tests pass for each league's pipeline

### Risk Factors
- **NCAA data quality** -- College sports have less reliable data sources than pro leagues. Mitigation: start with pro leagues, use CFBD API for football (most reliable), accept lower accuracy for NCAA Baseball initially.
- **Model quality per sport** -- Each sport has unique characteristics; a single modeling approach may not work equally well. Mitigation: sport-specific feature engineering and model tuning per league.
- **Scope** -- 5 new leagues is a lot of work. Mitigation: each league expansion is a self-contained unit; can ship incrementally.

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

**Live/In-Game Betting:**
6. Integrate SharpAPI SSE stream in lines-service for real-time line updates
7. Implement live simulation updates in simulation-engine: accept current game state, produce updated distributions
8. Implement live edge detection in agent with time-sensitivity alerts
9. Build live dashboard in UI with auto-updating odds and edges
10. Add live bet tracking and grading to bookie-emulator

**Parlay Correlation Analysis:**
11. Implement correlation modeling in simulation-engine: measure dependence between legs from the same game
12. Implement true parlay probability calculation in prediction-engine (accounting for correlation)
13. Detect mispriced correlated parlays (same-game parlays where sportsbooks assume independence)
14. Add parlay builder to UI

**Advanced ML:**
15. Implement ensemble methods in prediction-engine: combine XGBoost with random forests and/or neural nets
16. Build model A/B testing framework: run multiple model versions in parallel, compare performance
17. Implement automated model retraining pipeline

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
- **Live data latency** -- In-game betting requires sub-second line updates; SSE reliability matters. Mitigation: implement reconnection logic, stale-data detection, and fallback polling.
- **Correlation modeling complexity** -- Accurately modeling dependence between parlay legs is statistically challenging. Mitigation: start with empirical correlation from simulation output (run simulation once, measure joint distributions directly).
- **Model overfitting** -- More complex models (neural nets) risk overfitting on limited sports data. Mitigation: strict train/test splits by season, cross-validation, monitor out-of-sample performance.

---

## Phase Summary

| Phase | Name | Services | Est. Effort | Cumulative Value |
|-------|------|----------|-------------|------------------|
| 0 | Repository Bootstrap & Tooling | All repos (scaffolded), infra-ops (extended) | M | Consistent structure and quality gates |
| 1 | Infrastructure & Data Foundation | infra-ops, statistics-service, lines-service | XL | Can fetch and store NBA data |
| 2 | Prediction Core | simulation-engine, prediction-engine, agent (partial) | XL | Can generate NBA predictions |
| 3 | First Interface & Paper Trading | bookie-emulator, cli, agent (complete) | L | Full NBA workflow from terminal |
| 4 | Agent Intelligence & MCP | agent (enhanced), mcp-server | M | LLM analysis, IDE integration |
| 5 | Dashboard | ui | L | Visual dashboard operational |
| 6 | Sport Expansion | All services modified | XL | All 6 leagues supported |
| 7 | Advanced Features | All services modified | XL | Full bet type coverage |
