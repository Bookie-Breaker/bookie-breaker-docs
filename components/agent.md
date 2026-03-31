# bookie-breaker-agent

## Purpose

The agent serves a dual role: it orchestrates the end-to-end prediction pipeline on a schedule (triggering data pulls, running simulations, identifying edges, generating alerts) and acts as an LLM-powered analyst that interprets predictions, writes natural language analysis, and answers user questions about bets. It uses the Anthropic API for all LLM capabilities.

## Responsibilities

- Owns the prediction pipeline schedule and orchestration: deciding when to pull data, run simulations, generate predictions, and scan for edges.
- Coordinates calls across services in the correct order (statistics-service -> simulation-engine -> prediction-engine -> lines-service for edge detection -> bookie-emulator for optional paper bets).
- Generates natural language analysis of predictions, edges, and betting opportunities using LLM capabilities.
- Answers user questions about specific bets, matchups, and system output via CLI, UI, or MCP.
- Manages alerting when new +EV edges are detected.
- Owns the Anthropic API integration and all prompt engineering.

## Non-Responsibilities

- Does NOT run simulations itself. Delegates to simulation-engine.
- Does NOT train or run ML models. Delegates to prediction-engine.
- Does NOT ingest or store raw statistics or lines data. Delegates to statistics-service and lines-service.
- Does NOT place or track paper bets. Delegates to bookie-emulator.
- Does NOT own any user-facing interface. CLI, UI, and MCP server are separate services that call the agent.
- Does NOT own infrastructure or deployment configuration. That belongs to infra-ops.

## Inputs

| Source | Data | Mechanism |
|---|---|---|
| statistics-service | Current and historical stats for teams/players | API call |
| simulation-engine | Outcome probability distributions | API call (response) |
| prediction-engine | Calibrated probabilities with contextual adjustments | API call (response) |
| lines-service | Current lines/odds, line movement data | API call |
| bookie-emulator | Paper trading performance metrics | API call |
| CLI / UI / MCP server | User questions, commands, configuration | API request to agent |
| Anthropic API | LLM responses for analysis and question answering | API call (response) |
| Schedule/cron | Timed triggers for pipeline runs | Internal scheduler |

## Outputs

| Destination | Data | Mechanism |
|---|---|---|
| simulation-engine | Requests to run simulations for specific matchups | API call |
| prediction-engine | Requests to generate calibrated predictions | API call |
| lines-service | Requests for current lines to compare against predictions | API call |
| bookie-emulator | Requests to place paper bets on detected edges | API call |
| CLI / UI / MCP server | Edges, analysis text, answers to questions, alerts | API response |
| Anthropic API | Prompts for analysis generation | API call |

## Dependencies

- **statistics-service** -- source of all statistical data for analysis context
- **simulation-engine** -- runs the Monte Carlo simulations the agent orchestrates
- **prediction-engine** -- produces the calibrated probabilities the agent uses for edge detection
- **lines-service** -- provides the market lines the agent compares predictions against
- **bookie-emulator** -- executes paper bets and provides performance data
- **Anthropic API** (external) -- powers all LLM analysis and natural language capabilities

## Dependents

- **bookie-breaker-cli** -- calls the agent for analysis, questions, and pipeline control
- **bookie-breaker-ui** -- calls the agent for analysis and edge data
- **bookie-breaker-mcp-server** -- exposes agent capabilities as MCP tools
