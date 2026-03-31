# bookie-breaker-statistics-service

## Purpose

Ingests, processes, stores, and serves historical and current sports statistics from external APIs. Serves as the system's source of truth for all statistical data -- team stats, player stats, game results, and seasonal aggregates -- across all 6 supported leagues.

## Responsibilities

- Ingests raw statistics from external sports data APIs on a regular schedule (game results, box scores, player stats, team stats).
- Normalizes data into a canonical format across different source APIs and sports.
- Stores historical and current statistics with appropriate granularity (game-level, season-level, career-level).
- Computes derived statistics and aggregates (rolling averages, per-game rates, advanced metrics like offensive/defensive ratings, pace, efficiency).
- Serves team stats, player stats, game results, and seasonal aggregates via API.
- Covers all 6 leagues: NFL, NBA, MLB, NCAA Football, NCAA Basketball, NCAA Baseball.
- Provides injury reports and roster information when available from source APIs.
- Provides final game scores for bet grading by the bookie-emulator.

## Non-Responsibilities

- Does NOT store or serve betting lines/odds. That belongs to lines-service.
- Does NOT run simulations or predictions. It provides the raw data that feeds those systems.
- Does NOT calculate implied probabilities or detect edges.
- Does NOT perform any ML modeling or feature engineering. The prediction-engine owns that.
- Does NOT generate analysis or natural language content.

## Inputs

| Source | Data | Mechanism |
|---|---|---|
| External sports data APIs | Game results, box scores, player stats, team stats, injury reports, schedules | Scheduled API polling |
| agent | Requests for specific stats, matchup data, contextual data | API call |
| simulation-engine | Requests for team/player parameters needed for simulation | API call |
| prediction-engine | Requests for contextual features (injuries, rest days, etc.) | API call |
| bookie-emulator | Requests for final game scores to grade bets | API call |
| CLI / UI / MCP server | Stats lookup requests | API call |

## Outputs

| Destination | Data | Mechanism |
|---|---|---|
| simulation-engine | Team ratings, player stats, pace/efficiency metrics, matchup data | API response |
| prediction-engine | Contextual features: injury data, rest days, travel, seasonal trends | API response |
| bookie-emulator | Final game scores and results for bet grading | API response |
| agent | Stats summaries, matchup context for LLM analysis | API response |
| CLI / UI | Stats tables, player/team profiles | API response |

## Dependencies

- **External sports data APIs** (external) -- source of all raw statistical data

## Dependents

- **bookie-breaker-simulation-engine** -- consumes stats to parameterize simulations
- **bookie-breaker-prediction-engine** -- consumes contextual stats as ML features
- **bookie-breaker-bookie-emulator** -- consumes final scores to grade bets
- **bookie-breaker-agent** -- consumes stats for analysis context
- **bookie-breaker-cli** -- displays stats directly
- **bookie-breaker-ui** -- renders stats dashboards
- **bookie-breaker-mcp-server** -- exposes stats as MCP tools
