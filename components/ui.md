# bookie-breaker-ui

## Purpose

Web dashboard providing visual interfaces for the BookieBreaker system. One of three equal first-class interfaces (alongside CLI and MCP server), offering dashboards for edges, predictions, simulation results, paper trading performance, and line movement charts.

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

| Source | Data | Mechanism |
|---|---|---|
| User | Interactions (clicks, filters, queries) | Browser events |
| agent | Edges, analysis text, pipeline status, alerts | API response |
| bookie-emulator | Paper bet history, performance metrics | API response |
| lines-service | Current lines, line movement time series | API response |
| simulation-engine | Outcome distributions for visualization | API response |
| prediction-engine | Calibrated probabilities, feature importance | API response |
| statistics-service | Team/player stats for display | API response |

## Outputs

| Destination | Data | Mechanism |
|---|---|---|
| User | Rendered dashboards, charts, tables, analysis text | Browser rendering |
| agent | User questions, filter/query parameters | API call |
| bookie-emulator | Paper bet placement requests | API call |
| lines-service | Line and odds lookup requests | API call |
| statistics-service | Stats lookup requests | API call |

## Dependencies

- **agent** -- for LLM analysis, edge detection results, pipeline status
- **bookie-emulator** -- for paper trading data and performance metrics
- **lines-service** -- for lines, odds, and line movement data
- **simulation-engine** -- for simulation result distributions
- **prediction-engine** -- for prediction data
- **statistics-service** -- for stats data

## Dependents

None. The UI is a leaf node -- it is a consumer of backend services and has no downstream dependents.
