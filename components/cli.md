# bookie-breaker-cli

## Purpose

Command-line interface for interacting with the BookieBreaker system. One of three equal first-class interfaces (alongside UI and MCP server), supporting all system operations including viewing edges, predictions, paper bets, performance metrics, and querying the LLM analyst.

## Responsibilities

- Provides a terminal-based interface for all system capabilities.
- Formats and displays edges, predictions, simulation results, paper bet history, and performance metrics in the terminal.
- Supports interactive queries to the agent's LLM analyst (e.g., "Why do you like the over in this game?").
- Enables paper bet placement and management from the command line.
- Provides commands for monitoring system health and pipeline status.
- Handles authentication and user configuration locally.

## Non-Responsibilities

- Does NOT contain business logic. All computation, analysis, and data processing happens in backend services.
- Does NOT store data persistently (beyond local config/credentials). It is a stateless client.
- Does NOT duplicate functionality that exists in backend services -- it calls their APIs.
- Does NOT own the API contracts. It consumes APIs defined by the backend services.
- Does NOT manage infrastructure or deployment. That belongs to infra-ops.

## Inputs

| Source             | Data                                         | Mechanism      |
| ------------------ | -------------------------------------------- | -------------- |
| User               | Commands, queries, configuration             | Terminal input |
| agent              | Analysis text, edge alerts, pipeline status  | API response   |
| bookie-emulator    | Paper bet confirmations, performance metrics | API response   |
| lines-service      | Current lines and odds                       | API response   |
| simulation-engine  | Simulation result summaries                  | API response   |
| prediction-engine  | Prediction summaries                         | API response   |
| statistics-service | Stats lookups                                | API response   |

## Outputs

| Destination        | Data                                                    | Mechanism       |
| ------------------ | ------------------------------------------------------- | --------------- |
| User               | Formatted tables, charts (ASCII), analysis text, alerts | Terminal output |
| agent              | User questions, pipeline commands                       | API call        |
| bookie-emulator    | Paper bet placement requests                            | API call        |
| lines-service      | Line lookup requests                                    | API call        |
| statistics-service | Stats lookup requests                                   | API call        |

## Dependencies

- **agent** -- for LLM analysis, edge detection results, pipeline orchestration
- **bookie-emulator** -- for paper trading operations and performance data
- **lines-service** -- for current and historical lines
- **simulation-engine** -- for simulation results
- **prediction-engine** -- for prediction results
- **statistics-service** -- for stats lookups

## Dependents

None. The CLI is a leaf node -- it is a consumer of backend services and has no downstream dependents.

---

## Requirements

### Functional Requirements

- **FR-001:** Provide commands to list detected edges with filtering by league, date, minimum edge size, market type, and staleness status (e.g., `bb edges --league NBA --min-edge 3.0 --today`).
- **FR-002:** Provide commands to view detailed edge breakdowns including LLM-generated analysis, prediction confidence, feature importance, and line movement context.
- **FR-003:** Provide commands to view current betting lines for any game, with filtering by sportsbook, market type, and side (e.g., `bb lines LAL@BOS --type spread`).
- **FR-004:** Provide commands to view line movement history for a game/market (e.g., `bb lines LAL@BOS --movement`).
- **FR-005:** Provide commands to view simulation results, including key distribution percentiles and win/cover/over probabilities (e.g., `bb sim LAL@BOS`).
- **FR-006:** Provide commands to view predictions with calibrated probabilities and confidence intervals.
- **FR-007:** Provide commands to place paper bets manually via the bookie-emulator (e.g., `bb bet LAL@BOS spread home 2u`).
- **FR-008:** Provide commands to view the paper bet ledger with filtering by league, bet type, result, and date range (e.g., `bb bets --league NFL --result win --last 30d`).
- **FR-009:** Provide commands to view performance metrics: ROI, win rate, CLV, calibration, and breakdowns by league/bet type (e.g., `bb performance --league NBA --breakdown market_type`).
- **FR-010:** Support interactive LLM analyst queries via the agent (e.g., `bb ask "Why do you like the over in Lakers-Celtics?"`).
- **FR-011:** Provide commands to trigger and monitor pipeline runs (e.g., `bb pipeline run --league NBA`, `bb pipeline status`).
- **FR-012:** Provide commands to check system health across all services (e.g., `bb health`).
- **FR-013:** Format all output for terminal readability: tables for structured data, ASCII charts for distributions, colored text for edge highlights and win/loss indicators.
- **FR-014:** Support a watch mode that subscribes to `edge.detected` events and displays real-time alerts in the terminal.
- **FR-015:** Store user configuration locally (default league preferences, edge thresholds, output format preferences) in a config file.

### Non-Functional Requirements

- **Latency:** Data lookup commands (lines, stats, bets): response displayed in < 2 seconds. Analytical queries via LLM: response displayed in < 10 seconds. Pipeline trigger acknowledgment: < 1 second (pipeline itself runs asynchronously).
- **Throughput:** Single-user tool. Handles one command at a time (sequential). Watch mode handles up to 10 edge alerts per minute for display.
- **Availability:** Available whenever the underlying backend services are available. No independent uptime requirement since it is a client-side tool. Provides clear error messages when backend services are unreachable.
- **Storage:** Local config file: < 10 KB. No persistent data storage; all data comes from backend services.

### Data Ownership

None. The CLI does not own any domain entities. It is a stateless consumer of data from backend services.

### APIs Exposed

None. The CLI does not expose an API. It is an interactive terminal application.

### APIs Consumed

| Service            | Endpoint                                 | Purpose                                   |
| ------------------ | ---------------------------------------- | ----------------------------------------- |
| agent              | `GET /api/v1/edges`                      | Fetch detected edges for display          |
| agent              | `GET /api/v1/edges/{edge_id}`            | Fetch detailed edge with analysis         |
| agent              | `POST /api/v1/query`                     | Submit analytical question to LLM analyst |
| agent              | `POST /api/v1/pipeline/run`              | Trigger a pipeline run                    |
| agent              | `GET /api/v1/pipeline/status`            | Check pipeline status                     |
| agent              | `GET /api/v1/health`                     | Check system health                       |
| agent              | `GET /api/v1/alerts`                     | Fetch recent alerts                       |
| bookie-emulator    | `POST /api/v1/bets`                      | Place a paper bet                         |
| bookie-emulator    | `GET /api/v1/bets`                       | Fetch bet ledger                          |
| bookie-emulator    | `GET /api/v1/performance`                | Fetch performance metrics                 |
| bookie-emulator    | `GET /api/v1/performance/calibration`    | Fetch calibration data                    |
| lines-service      | `GET /api/v1/lines/{game_id}`            | Fetch current lines for a game            |
| lines-service      | `GET /api/v1/lines/{game_id}/movement`   | Fetch line movement history               |
| statistics-service | `GET /api/v1/teams/{team_id}/stats`      | Fetch team stats                          |
| statistics-service | `GET /api/v1/players/{player_id}/stats`  | Fetch player stats                        |
| simulation-engine  | `GET /api/v1/simulations/game/{game_id}` | Fetch simulation results                  |
| prediction-engine  | `GET /api/v1/predictions/game/{game_id}` | Fetch predictions                         |

### Events Published

None. The CLI does not publish events.

### Events Subscribed

| Event           | Channel                | Purpose                                                        |
| --------------- | ---------------------- | -------------------------------------------------------------- |
| `edge.detected` | `events:edge.detected` | In watch mode, displays real-time edge alerts in the terminal. |

### Storage Requirements

- **Database:** None. The CLI is stateless.
- **Local config file:** YAML or TOML file in `~/.config/bookiebreaker/config.yaml` storing user preferences (default league, edge threshold, output format, service URLs).
- **No persistent data storage:** All data is fetched from backend services on demand.
