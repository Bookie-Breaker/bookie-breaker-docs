# bookie-breaker-ui

## Purpose

Web dashboard providing visual interfaces for the BookieBreaker system. One of three equal first-class interfaces
(alongside CLI and MCP server), offering dashboards for edges, predictions, simulation results, paper trading
performance, and line movement charts.

## Responsibilities

- Renders visual dashboards for detected +EV edges with filtering and sorting.
- Displays prediction summaries with calibrated probabilities, confidence intervals, and feature importance.
- Visualizes simulation output distributions (histograms, probability curves, percentile breakdowns).
- Renders paper trading performance dashboards: ROI over time, win rate, CLV charts, calibration plots, bet ledger.
- Displays line movement charts showing how lines have moved from open to current.
- Provides data exploration tools: filtering by sport, league, date, bet type, edge size.
- Supports at-a-glance monitoring of system status and recent pipeline runs.
- Provides interactive query interface to the agent's LLM analyst.

## Non-Responsibilities

- Does NOT contain business logic. All computation happens in backend services.
- Does NOT store data persistently (beyond browser-local preferences). It is a stateless client.
- Does NOT duplicate backend logic for calculations, filtering, or aggregation -- it calls APIs.
- Does NOT own API contracts. It consumes APIs defined by backend services.
- Does NOT manage infrastructure or deployment.
- Does NOT provide a mobile-native experience (web-responsive only, not a native app).

## Inputs

| Source             | Data                                          | Mechanism      |
| ------------------ | --------------------------------------------- | -------------- |
| User               | Interactions (clicks, filters, queries)       | Browser events |
| agent              | Edges, analysis text, pipeline status, alerts | API response   |
| bookie-emulator    | Paper bet history, performance metrics        | API response   |
| lines-service      | Current lines, line movement time series      | API response   |
| simulation-engine  | Outcome distributions for visualization       | API response   |
| prediction-engine  | Calibrated probabilities, feature importance  | API response   |
| statistics-service | Team/player stats for display                 | API response   |

## Outputs

| Destination        | Data                                               | Mechanism         |
| ------------------ | -------------------------------------------------- | ----------------- |
| User               | Rendered dashboards, charts, tables, analysis text | Browser rendering |
| agent              | User questions, filter/query parameters            | API call          |
| bookie-emulator    | Paper bet placement requests                       | API call          |
| lines-service      | Line and odds lookup requests                      | API call          |
| statistics-service | Stats lookup requests                              | API call          |

## Dependencies

- **agent** -- for LLM analysis, edge detection results, pipeline status
- **bookie-emulator** -- for paper trading data and performance metrics
- **lines-service** -- for lines, odds, and line movement data
- **simulation-engine** -- for simulation result distributions
- **prediction-engine** -- for prediction data
- **statistics-service** -- for stats data

## Dependents

None. The UI is a leaf node -- it is a consumer of backend services and has no downstream dependents.

---

## Requirements

### Functional Requirements

- **FR-001:** Render an edges dashboard showing all detected +EV edges with sortable columns: edge size, league, game,
  bet type, predicted probability, implied probability, best odds, sportsbook, and detection time. Support filtering by
  league, date, minimum edge size, market type, and staleness.
- **FR-002:** Display detailed edge views with LLM-generated analysis, prediction confidence intervals, feature
  importance breakdown, and line movement chart for the relevant market.
- **FR-003:** Render line movement charts showing how lines have moved from open to current for any
  game/sportsbook/market, with time on the x-axis and line value/odds on the y-axis.
- **FR-004:** Display current betting lines for any game across all tracked sportsbooks, with best-odds highlighting.
- **FR-005:** Visualize simulation output distributions as histograms or probability density curves for score
  distributions, margin distributions, and total distributions. Show key percentiles (10th, 25th, 50th, 75th, 90th).
- **FR-006:** Render prediction summaries with calibrated probabilities, confidence intervals, and feature importance
  bar charts.
- **FR-007:** Render paper trading performance dashboards: ROI over time (line chart), cumulative units won/lost, win
  rate, average CLV, longest streaks, and calibration plots.
- **FR-008:** Display the paper bet ledger with filtering, sorting, and pagination. Show individual bet details
  including placement odds, closing odds, CLV, and P/L.
- **FR-009:** Provide performance breakdown views: ROI/win rate by league, by market type, by sportsbook, and by time
  period.
- **FR-010:** Provide an interactive query interface (chat-like) for the agent's LLM analyst, displaying formatted
  markdown responses.
- **FR-011:** Subscribe to Redis `edge.detected` and `prediction.completed` events for real-time dashboard updates
  without requiring manual page refresh.
- **FR-012:** Display system status and pipeline health: last run time, services status, and recent pipeline run
  summaries.
- **FR-013:** Provide data exploration tools: filtering by sport, league, date range, bet type, and edge size,
  applicable across all views.
- **FR-014:** Support responsive web layout for desktop and tablet screen sizes. Mobile is not required.
- **FR-015:** Support placing paper bets directly from edge detail views via the bookie-emulator API.

### Non-Functional Requirements

- **Latency:** Initial page load (dashboard): < 3 seconds. Data refresh on filter/sort change: < 1 second. Real-time
  event updates: reflected in UI within 2 seconds of event publication. LLM analyst response: streaming display begins
  within 3 seconds.
- **Throughput:** Support 1-3 concurrent browser sessions (solo developer project). Handle up to 10 real-time event
  updates per minute for live dashboard refresh.
- **Availability:** Available whenever the underlying backend services are available. The UI itself is a static web
  application served from a container, so it is up whenever its container runs. Graceful degradation: if a backend
  service is unreachable, display cached data with a stale-data indicator rather than blank panels.
- **Storage:** Browser localStorage for user preferences (theme, default filters, dashboard layout): < 100 KB. No
  server-side persistent storage.

### Data Ownership

None. The UI does not own any domain entities. It is a stateless consumer of data from backend services.

### APIs Exposed

None. The UI does not expose an API to other services. It serves a web application to browsers on port 3000.

### APIs Consumed

All calls happen server-side (SvelteKit load functions and an allowlist of `/api` proxy routes per
[ADR-025](../decisions/025-ui-server-proxy-and-sse-bridge.md)); the browser never talks to backend services
directly. Paths below are the canonical api-contracts paths.

| Service            | Endpoint                                                    | Purpose                                     |
| ------------------ | ----------------------------------------------------------- | ------------------------------------------- |
| agent              | `GET /api/v1/agent/edges`                                   | Fetch edges for dashboard                   |
| agent              | `GET /api/v1/agent/edges/{edge_id}`                         | Fetch detailed edge with analysis           |
| agent              | `GET /api/v1/agent/slate`                                   | Today's games with predictions and edges    |
| agent              | `GET /api/v1/agent/dashboard`                               | Home page summary aggregate                 |
| agent              | `POST /api/v1/agent/analysis/stream`                        | Streaming LLM analyst chat (SSE, ADR-024)   |
| agent              | `GET /api/v1/agent/analysis/{analysis_id}`                  | Fetch a stored LLM analysis                 |
| agent              | `GET /api/v1/agent/alerts` + `PUT /alerts/{id}/acknowledge` | Alert list and acknowledgement              |
| agent              | `POST /api/v1/agent/pipeline/run`                           | Trigger pipeline runs                       |
| agent              | `GET /api/v1/agent/pipeline/runs/{pipeline_run_id}`         | Check pipeline run status                   |
| bookie-emulator    | `POST /api/v1/emulator/bets`                                | Place paper bets from edge views            |
| bookie-emulator    | `GET /api/v1/emulator/bets` + `GET /bets/{bet_id}`          | Bet ledger and detail                       |
| bookie-emulator    | `GET /api/v1/emulator/performance`                          | Performance metrics                         |
| bookie-emulator    | `GET /api/v1/emulator/performance/calibration`              | Calibration bins for the reliability plot   |
| bookie-emulator    | `GET /api/v1/emulator/performance/breakdown`                | Performance breakdowns                      |
| bookie-emulator    | `GET /api/v1/emulator/bankroll/history`                     | Bankroll/ROI time series for charting       |
| lines-service      | `GET /api/v1/lines/current`                                 | Current lines grid                          |
| lines-service      | `GET /api/v1/lines/game/{game_id}/movement`                 | Line movement data for charts               |
| statistics-service | `GET /api/v1/stats/...`                                     | Team/player stats and schedules (as needed) |
| simulation-engine  | `GET /api/v1/sim/games/{game_id}/latest`                    | Latest simulation run pointer               |
| simulation-engine  | `GET /api/v1/sim/simulations/{simulation_id}/distributions` | Distribution histograms for charts          |
| prediction-engine  | `GET /api/v1/predict/games/{game_id}/latest`                | Predictions (mostly via agent edge detail)  |

### Events Published

None. The UI does not publish events.

### Events Subscribed

The SvelteKit server holds one shared Redis subscriber and bridges events to browsers as SSE
(`GET /api/events`); payloads are never rendered — pages refetch via REST on receipt.

| Event                  | Channel                       | Purpose                                                  |
| ---------------------- | ----------------------------- | -------------------------------------------------------- |
| `edge.detected`        | `events:edge.detected`        | Display real-time edge alerts; refresh edges/dashboard.  |
| `prediction.completed` | `events:prediction.completed` | Refresh edge display when new predictions are available. |
| `lines.updated`        | `events:lines.updated`        | Refresh the lines grid and slate.                        |
| `game.completed`       | `events:game.completed`       | Refresh slate and dashboard on final scores.             |
| `bet.graded`           | `events:bet.graded`           | Live ledger/performance refresh when bets grade.         |

### Storage Requirements

- **Database:** None. The UI has no server-side persistent storage.
- **Browser localStorage:** User preferences (theme, default league filter, dashboard layout, recent queries): < 100 KB.
- **Static assets:** The UI container serves bundled JavaScript, CSS, and image assets. Total bundle size target: < 5
  MB.
