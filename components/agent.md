# bookie-breaker-agent

## Purpose

The agent serves a dual role: it orchestrates the end-to-end prediction pipeline on a schedule (triggering data pulls,
running simulations, identifying edges, generating alerts) and acts as an LLM-powered analyst that interprets
predictions, writes natural language analysis, and answers user questions about bets. It uses the Anthropic API for all
LLM capabilities.

## Responsibilities

- Owns the prediction pipeline schedule and orchestration: deciding when to pull data, run simulations, generate
  predictions, and scan for edges.
- Coordinates calls across services in the correct order (statistics-service -> simulation-engine -> prediction-engine
  -> lines-service for edge detection -> bookie-emulator for optional paper bets).
- Generates natural language analysis of predictions, edges, and betting opportunities using LLM capabilities.
- Answers user questions about specific bets, matchups, and system output via CLI, UI, or MCP.
- Manages alerting when new +EV edges are detected.
- Owns the LLM provider abstraction layer: configurable backend supporting Anthropic API (cloud) and local LLM via
  Ollama (self-hosted). Switching providers is config-only (`LLM_PROVIDER=anthropic|ollama`, `LLM_BASE_URL`). See
  [ADR-011](../decisions/011-local-llm-strategy.md).
- Owns all prompt engineering.

## Non-Responsibilities

- Does NOT run simulations itself. Delegates to simulation-engine.
- Does NOT train or run ML models. Delegates to prediction-engine.
- Does NOT ingest or store raw statistics or lines data. Delegates to statistics-service and lines-service.
- Does NOT place or track paper bets. Delegates to bookie-emulator.
- Does NOT own any user-facing interface. CLI, UI, and MCP server are separate services that call the agent.
- Does NOT own infrastructure or deployment configuration. That belongs to infra-ops.

## Inputs

| Source                                 | Data                                                 | Mechanism            |
| -------------------------------------- | ---------------------------------------------------- | -------------------- |
| statistics-service                     | Current and historical stats for teams/players       | API call             |
| simulation-engine                      | Outcome probability distributions                    | API call (response)  |
| prediction-engine                      | Calibrated probabilities with contextual adjustments | API call (response)  |
| lines-service                          | Current lines/odds, line movement data               | API call             |
| bookie-emulator                        | Paper trading performance metrics                    | API call             |
| CLI / UI / MCP server                  | User questions, commands, configuration              | API request to agent |
| LLM provider (Anthropic API or Ollama) | LLM responses for analysis and question answering    | API call (response)  |
| Schedule/cron                          | Timed triggers for pipeline runs                     | Internal scheduler   |

## Outputs

| Destination                            | Data                                                      | Mechanism    |
| -------------------------------------- | --------------------------------------------------------- | ------------ |
| simulation-engine                      | Requests to run simulations for specific matchups         | API call     |
| prediction-engine                      | Requests to generate calibrated predictions               | API call     |
| lines-service                          | Requests for current lines to compare against predictions | API call     |
| bookie-emulator                        | Requests to place paper bets on detected edges            | API call     |
| CLI / UI / MCP server                  | Edges, analysis text, answers to questions, alerts        | API response |
| LLM provider (Anthropic API or Ollama) | Prompts for analysis generation                           | API call     |

## Dependencies

- **statistics-service** -- source of all statistical data for analysis context
- **simulation-engine** -- runs the Monte Carlo simulations the agent orchestrates
- **prediction-engine** -- produces the calibrated probabilities the agent uses for edge detection
- **lines-service** -- provides the market lines the agent compares predictions against
- **bookie-emulator** -- executes paper bets and provides performance data
- **LLM provider** (external) -- powers all LLM analysis and natural language capabilities (Anthropic API for cloud,
  Ollama for self-hosted local models)

## Dependents

- **bookie-breaker-cli** -- calls the agent for analysis, questions, and pipeline control
- **bookie-breaker-ui** -- calls the agent for analysis and edge data
- **bookie-breaker-mcp-server** -- exposes agent capabilities as MCP tools

---

## Requirements

### Functional Requirements

- **FR-001:** Run the full prediction pipeline on a configurable schedule: determine which games need predictions
  (upcoming games within a configurable lookahead window, e.g., 24-48 hours), trigger simulations, collect predictions,
  compare against market lines, and detect edges.
- **FR-002:** Support event-driven pipeline re-runs: when `lines.updated` or `stats.updated` events indicate material
  new data for games with existing predictions, re-evaluate whether to re-run the pipeline for affected games.
- **FR-003:** Orchestrate the pipeline in the correct sequence: statistics-service (params) -> simulation-engine
  (distributions) -> prediction-engine (calibrated probabilities) -> lines-service (current market lines) -> edge
  detection -> bookie-emulator (paper bets).
- **FR-004:** Perform edge detection by comparing calibrated predicted probabilities from prediction-engine against
  market-implied probabilities from lines-service. Identify edges where predicted probability exceeds implied
  probability by a configurable threshold (default >= 3%).
- **FR-005:** Calculate Kelly criterion recommended stake size for each detected edge and apply a configurable
  fractional Kelly multiplier (default 1/4 Kelly).
- **FR-006:** Automatically place paper bets via bookie-emulator when edges exceed the configured threshold, including
  game, bet type, odds, stake, predicted probability, and edge size.
- **FR-007:** Generate natural language analysis of detected edges using the Anthropic API, including reasoning about
  why the edge exists, key statistical factors, and risk considerations.
- **FR-008:** Answer user questions about specific bets, matchups, edges, and system output by gathering relevant data
  from backend services and synthesizing responses via the Anthropic API.
- **FR-009:** Route all analytical queries from CLI, UI, and MCP server through a unified API, determining which backend
  services to call based on the query type.
- **FR-010:** Manage alerting for detected edges: publish `edge.detected` events to Redis and deliver alerts to
  configured channels (CLI, UI, MCP) with appropriate priority levels (HIGH for large edges near game time, MEDIUM for
  moderate edges, LOW for informational).
- **FR-011:** Persist detected edges with full context: prediction ID, betting line ID, edge percentage, expected value,
  recommended stake, detection timestamp, and staleness tracking.
- **FR-012:** Mark edges as stale when subsequent lines updates indicate the line has moved since detection.
- **FR-013:** Persist LLM-generated analyses (Analysis entities) with metadata about the model used and input data
  provided.
- **FR-014:** Expose pipeline status, last run timestamps, and system health summary to interfaces.
- **FR-015:** Support league-aware scheduling: skip polling and pipeline runs for leagues in their offseason based on
  League season metadata.

### Non-Functional Requirements

- **Latency:** Full daily pipeline (all games for a day): 5-30 minutes end-to-end. Single game re-evaluation: 30 seconds
  to 2 minutes. Edge detection (comparison step after predictions and lines are available): < 1 second. Alert delivery
  after edge detection: < 30 seconds. Data query responses (lines, stats, performance): < 2 seconds. Analytical
  questions (requiring LLM): 3-10 seconds.
- **Throughput:** Process up to 30 games per daily pipeline run. Handle up to 20 concurrent user queries from
  interfaces. Make up to 100 parallel API calls to backend services during pipeline orchestration.
- **Availability:** 99% uptime. Graceful degradation: if a backend service is temporarily unavailable, return partial
  results with a warning rather than failing entirely. Scheduled pipeline runs that fail should be retried automatically
  with exponential backoff (max 3 retries).
- **Storage:** Edges: ~10,000-50,000 edges/year (~2 KB each). EdgeAlerts: ~10,000-50,000/year. Analyses:
  ~5,000-20,000/year (~5-10 KB each, mostly LLM-generated text). Query/response history: optional, session-scoped.
  Total: ~500 MB - 1 GB/year.

### Data Ownership

This service is the source of truth for:

- **Edge** -- detected betting edges with predicted vs. implied probabilities, edge size, expected value, Kelly
  fraction, and staleness tracking.
- **EdgeAlert** -- notifications generated for detected edges, tracked by channel, priority, and delivery/acknowledgment
  status.
- **Analysis** -- LLM-generated narrative analyses of games, edges, and performance periods.

### APIs Exposed

| Method + Path                               | Description                                            | Key Query Parameters                                    | Consumers               |
| ------------------------------------------- | ------------------------------------------------------ | ------------------------------------------------------- | ----------------------- |
| `POST /api/v1/pipeline/run`                 | Trigger a pipeline run for specified games or leagues  | Body: league, game_ids, force_refresh                   | CLI, UI, MCP            |
| `GET /api/v1/pipeline/status`               | Get status of current/last pipeline run                | --                                                      | CLI, UI, MCP            |
| `GET /api/v1/edges`                         | List detected edges                                    | `league`, `date`, `min_edge`, `market_type`, `is_stale` | CLI, UI, MCP            |
| `GET /api/v1/edges/{edge_id}`               | Get detailed edge with analysis                        | --                                                      | CLI, UI, MCP            |
| `POST /api/v1/query`                        | Submit an analytical question for LLM-powered response | Body: question, context (game_id, league, etc.)         | CLI, UI, MCP            |
| `GET /api/v1/alerts`                        | List recent alerts                                     | `priority`, `channel`, `acknowledged`                   | CLI, UI, MCP            |
| `PUT /api/v1/alerts/{alert_id}/acknowledge` | Acknowledge an alert                                   | --                                                      | CLI, UI, MCP            |
| `GET /api/v1/analyses/{analysis_id}`        | Get a specific analysis                                | --                                                      | CLI, UI, MCP            |
| `GET /api/v1/analyses`                      | List analyses                                          | `analysis_type`, `game_id`, `date`                      | CLI, UI, MCP            |
| `GET /api/v1/health`                        | System health summary (all services)                   | --                                                      | CLI, UI, MCP, infra-ops |

### APIs Consumed

| Service                 | Endpoint                                                     | Purpose                                        |
| ----------------------- | ------------------------------------------------------------ | ---------------------------------------------- |
| simulation-engine       | `POST /api/v1/simulations`                                   | Trigger batch simulation runs                  |
| simulation-engine       | `GET /api/v1/simulations/game/{game_id}`                     | Get simulation results                         |
| prediction-engine       | `POST /api/v1/predictions`                                   | Trigger batch prediction generation            |
| prediction-engine       | `GET /api/v1/predictions/game/{game_id}`                     | Get calibrated probabilities                   |
| lines-service           | `GET /api/v1/lines/{game_id}`                                | Get current market lines for edge detection    |
| lines-service           | `GET /api/v1/lines/{game_id}/best`                           | Get best available line per market             |
| statistics-service      | `GET /api/v1/games`                                          | Get upcoming games to determine pipeline scope |
| statistics-service      | `GET /api/v1/teams/{team_id}/stats`                          | Get stats for LLM analysis context             |
| statistics-service      | `GET /api/v1/leagues`                                        | Get league season metadata for scheduling      |
| bookie-emulator         | `POST /api/v1/bets`                                          | Place paper bets on detected edges             |
| bookie-emulator         | `GET /api/v1/performance`                                    | Get performance metrics for analysis           |
| LLM provider (external) | `POST /v1/messages` (Anthropic) or `POST /api/chat` (Ollama) | Generate natural language analysis             |

### Events Published

| Event                  | Channel                       | Description                                                                                                                                          |
| ---------------------- | ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `prediction.completed` | `events:prediction.completed` | Published after edge detection completes for a batch. Payload includes batch_id, game_ids, league, edges_found count.                                |
| `edge.detected`        | `events:edge.detected`        | Published for each actionable edge. Payload includes game_id, league, bet_type, side, edge_pct, predicted_prob, implied_prob, best_odds, sportsbook. |

### Events Subscribed

| Event                  | Channel                       | Purpose                                                                                                                                            |
| ---------------------- | ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `lines.updated`        | `events:lines.updated`        | React to new/changed lines by evaluating whether to re-run predictions for affected games. Also marks existing edges as stale if lines have moved. |
| `stats.updated`        | `events:stats.updated`        | React to new stats by evaluating whether to re-run the pipeline for affected games.                                                                |
| `simulation.completed` | `events:simulation.completed` | Informational; used for monitoring and logging pipeline progress.                                                                                  |
| `edge.detected`        | `events:edge.detected`        | Listens to its own edge events for self-monitoring (e.g., counting edges per day).                                                                 |

### Storage Requirements

- **Database:** PostgreSQL.
- **Key tables:**
  - `edges` -- detected betting edges (~10K-50K rows/year).
  - `edge_alerts` -- alert records (~10K-50K rows/year).
  - `analyses` -- LLM-generated analyses (~5K-20K rows/year, content as TEXT).
  - `pipeline_runs` -- pipeline execution log (~2K-5K rows/year).
  - `query_log` -- optional user query history (~10K-50K rows/year).
- **Estimated row counts:** Year 1: ~30K-150K rows total. Year 3: ~100K-500K rows.
- **Growth:** ~100-500 new edge rows/day during active seasons.
- **Indexing priorities:**
  1. `edges(game_id, market_type, detected_at DESC)` -- edge lookup by game.
  2. `edges(detected_at DESC, is_stale)` -- recent active edges for display.
  3. `edge_alerts(channel, priority, delivered_at)` -- alert delivery queries.
  4. `analyses(game_id, analysis_type)` -- analysis lookup.
- **Redis:** Used for event deduplication (1h TTL), query response caching (5m TTL), and pipeline state tracking.
