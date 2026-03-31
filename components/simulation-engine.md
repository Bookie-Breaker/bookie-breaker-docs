# bookie-breaker-simulation-engine

## Purpose

The intellectual core of BookieBreaker. Runs Monte Carlo simulations to generate base probability distributions for game outcomes using a sport-agnostic framework with sport-specific simulation plugins. Produces full outcome distributions -- not point predictions -- that feed into the prediction-engine for contextual adjustment.

## Responsibilities

- Owns the Monte Carlo simulation framework: running N iterations of a game simulation and aggregating results into outcome distributions.
- Owns the sport-agnostic simulation interface that all sport plugins implement.
- Owns sport-specific simulation plugins, each modeling the sport's mechanics faithfully:
  - **Football (NFL, NCAA FB):** Discrete play-by-play simulation (drive-level or play-level).
  - **Basketball (NBA, NCAA BB):** Continuous flow / possession-based simulation.
  - **Baseball (MLB, NCAA Baseball):** Pitcher-batter matchup simulation with plate appearance resolution.
- Produces outcome distributions: full score distributions, margin distributions, total distributions, and derived prop distributions.
- Manages simulation parameters (number of iterations, convergence criteria, random seeds for reproducibility).
- Provides simulation metadata: convergence diagnostics, variance estimates, distribution shapes.

## Non-Responsibilities

- Does NOT apply ML adjustments for contextual factors (injuries, weather, rest). That belongs to the prediction-engine.
- Does NOT compare predictions against market lines or detect edges. The agent does that.
- Does NOT ingest or store raw statistics. It requests the data it needs from the statistics-service.
- Does NOT generate natural language analysis of results. The agent handles that.
- Does NOT place or track bets.
- Does NOT train ML models. It uses statistical/mathematical models of sport mechanics, not learned models.

## Inputs

| Source | Data | Mechanism |
|---|---|---|
| statistics-service | Team and player statistics needed for simulation parameters (offensive/defensive ratings, pace, efficiency, pitcher stats, batting stats, etc.) | API call |
| agent | Requests to simulate specific matchups with given parameters | API call |

## Outputs

| Destination | Data | Mechanism |
|---|---|---|
| prediction-engine | Raw outcome distributions (score distributions, margin distributions, total distributions, prop-relevant distributions) | API response |
| agent | Simulation summaries (may be accessed directly for display) | API response |
| CLI / UI / MCP server | Simulation result visualizations (distribution charts, key percentiles) | API response |

## Dependencies

- **statistics-service** -- provides all team and player data that parameterize the simulations

## Dependents

- **bookie-breaker-prediction-engine** -- consumes simulation distributions as input for ML adjustment
- **bookie-breaker-agent** -- orchestrates simulation runs and may display raw results
- **bookie-breaker-cli** -- may query simulation results directly
- **bookie-breaker-ui** -- may render simulation distributions directly
- **bookie-breaker-mcp-server** -- may expose simulation results as MCP tools
