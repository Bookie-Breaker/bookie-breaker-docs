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

| Source | Data | Mechanism |
|---|---|---|
| User | Commands, queries, configuration | Terminal input |
| agent | Analysis text, edge alerts, pipeline status | API response |
| bookie-emulator | Paper bet confirmations, performance metrics | API response |
| lines-service | Current lines and odds | API response |
| simulation-engine | Simulation result summaries | API response |
| prediction-engine | Prediction summaries | API response |
| statistics-service | Stats lookups | API response |

## Outputs

| Destination | Data | Mechanism |
|---|---|---|
| User | Formatted tables, charts (ASCII), analysis text, alerts | Terminal output |
| agent | User questions, pipeline commands | API call |
| bookie-emulator | Paper bet placement requests | API call |
| lines-service | Line lookup requests | API call |
| statistics-service | Stats lookup requests | API call |

## Dependencies

- **agent** -- for LLM analysis, edge detection results, pipeline orchestration
- **bookie-emulator** -- for paper trading operations and performance data
- **lines-service** -- for current and historical lines
- **simulation-engine** -- for simulation results
- **prediction-engine** -- for prediction results
- **statistics-service** -- for stats lookups

## Dependents

None. The CLI is a leaf node -- it is a consumer of backend services and has no downstream dependents.
