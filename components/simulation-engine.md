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

| Source             | Data                                                                                                                                            | Mechanism |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------- | --------- |
| statistics-service | Team and player statistics needed for simulation parameters (offensive/defensive ratings, pace, efficiency, pitcher stats, batting stats, etc.) | API call  |
| agent              | Requests to simulate specific matchups with given parameters                                                                                    | API call  |

## Outputs

| Destination           | Data                                                                                                                    | Mechanism    |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------- | ------------ |
| prediction-engine     | Raw outcome distributions (score distributions, margin distributions, total distributions, prop-relevant distributions) | API response |
| agent                 | Simulation summaries (may be accessed directly for display)                                                             | API response |
| CLI / UI / MCP server | Simulation result visualizations (distribution charts, key percentiles)                                                 | API response |

## Dependencies

- **statistics-service** -- provides all team and player data that parameterize the simulations

## Dependents

- **bookie-breaker-prediction-engine** -- consumes simulation distributions as input for ML adjustment
- **bookie-breaker-agent** -- orchestrates simulation runs and may display raw results
- **bookie-breaker-cli** -- may query simulation results directly
- **bookie-breaker-ui** -- may render simulation distributions directly
- **bookie-breaker-mcp-server** -- may expose simulation results as MCP tools

---

## Requirements

### Functional Requirements

- **FR-001:** Run Monte Carlo simulations with configurable iteration counts (default 10,000; up to 50,000 for high-stakes or high-variance matchups) for any game across all 6 leagues.
- **FR-002:** Implement a sport-agnostic simulation framework with a plugin interface that all sport-specific simulation plugins implement.
- **FR-003:** Implement a football simulation plugin (NFL, NCAA_FB) that models discrete play-by-play or drive-level game flow using team offensive/defensive ratings, pace, scoring distributions, and turnover rates.
- **FR-004:** Implement a basketball simulation plugin (NBA, NCAA_BB) that models continuous possession-based game flow using pace, offensive/defensive efficiency, scoring distributions, and foul rates.
- **FR-005:** Implement a baseball simulation plugin (MLB, NCAA_BSB) that models pitcher-batter matchups with plate appearance resolution, using pitcher stats, batting stats, park factors, and bullpen usage patterns.
- **FR-006:** Produce full outcome distributions from each simulation run: home score distribution, away score distribution, margin distribution, total distribution, home/away win probabilities, and draw probability (for regulation-only markets).
- **FR-007:** Pre-compute spread cover probabilities at common line values (e.g., -3.5, -7, +3.5, +7) and total over/under probabilities at common totals (e.g., 44.5, 45.5, 200.5, 210.5) to avoid repeated CDF calculations downstream.
- **FR-008:** Request team and player statistics from statistics-service to parameterize each simulation (offensive/defensive ratings, pace, efficiency, pitcher stats, batting stats, etc.).
- **FR-009:** Support simulation caching: detect when a game has already been simulated with identical input parameters (via parameters_hash) and return cached results instead of re-running.
- **FR-010:** Support forced re-runs when the agent signals that input data has been updated since the last simulation.
- **FR-011:** Provide convergence diagnostics: detect when distributions have stabilized and optionally stop early if convergence_threshold is met before max iterations.
- **FR-012:** Support reproducible simulations via configurable random seeds.
- **FR-013:** Support batch simulation requests: simulate multiple games in a single API call, processing them in parallel.
- **FR-014:** Publish a `simulation.completed` event to `events:simulation.completed` when a batch of simulations finishes, including batch_id, game_ids, league, iterations_per_game, and duration.
- **FR-015:** Log simulation metadata (SimulationRun) for every execution, including config, timing, convergence status, and parameters_hash.

### Non-Functional Requirements

- **Latency:** Single game simulation (10,000 iterations): < 30 seconds. Single game simulation (50,000 iterations): < 2 minutes. Full daily batch (all games for a day, typically 5-30 games): 5-15 minutes.
- **Throughput:** Process up to 30 simultaneous game simulations in parallel during daily batch runs. Handle up to 10 ad-hoc single-game simulation requests/minute from the agent.
- **Availability:** 99% uptime. Graceful degradation: if statistics-service is temporarily unavailable, return an error with retry guidance rather than producing simulations with stale data. Cached results remain available during outages.
- **Storage:** Simulation results are relatively compact per run (~5-10 KB per SimulationResult). Estimated 50,000-100,000 simulation runs/year. Total storage: ~500 MB - 1 GB/year for results. SimulationRun metadata adds ~100 MB/year.

### Data Ownership

This service is the source of truth for:

- **SimulationConfig** -- default and custom simulation configurations per sport.
- **SimulationRun** -- metadata about each simulation execution (timing, convergence, parameters hash).
- **SimulationResult** -- output distributions, probabilities, and percentiles from each simulation run.

### APIs Exposed

| Method + Path                                          | Description                                   | Key Query Parameters                                | Consumers                              |
| ------------------------------------------------------ | --------------------------------------------- | --------------------------------------------------- | -------------------------------------- |
| `POST /api/v1/simulations`                             | Run simulation(s) for one or more games       | Body: game_ids, config overrides (iterations, seed) | agent                                  |
| `GET /api/v1/simulations/{simulation_run_id}`          | Get results of a specific simulation run      | --                                                  | agent, prediction-engine, CLI, UI, MCP |
| `GET /api/v1/simulations/game/{game_id}`               | Get most recent simulation results for a game | `config_id`, `force_refresh`                        | prediction-engine, agent, CLI, UI, MCP |
| `GET /api/v1/simulations/game/{game_id}/distributions` | Get raw distribution data for visualization   | `distribution_type` (margin, total, scores)         | UI (charts), CLI                       |
| `GET /api/v1/simulations/batch/{batch_id}`             | Get status and results of a batch simulation  | --                                                  | agent                                  |
| `GET /api/v1/configs`                                  | List available simulation configs             | `sport`                                             | agent                                  |
| `PUT /api/v1/configs/{config_id}`                      | Update a simulation config                    | --                                                  | agent                                  |

### APIs Consumed

| Service            | Endpoint                                            | Purpose                                                                                          |
| ------------------ | --------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| statistics-service | `GET /api/v1/teams/{team_id}/stats`                 | Get team offensive/defensive ratings, pace, efficiency for simulation parameters                 |
| statistics-service | `GET /api/v1/players/{player_id}/stats`             | Get player stats for pitcher-batter matchups (baseball), key player impact (basketball/football) |
| statistics-service | `GET /api/v1/matchup/{home_team_id}/{away_team_id}` | Get matchup context for head-to-head history                                                     |
| statistics-service | `GET /api/v1/venues/{venue_id}`                     | Get venue details for park factors (baseball), dome/surface (football)                           |

### Events Published

| Event                  | Channel                       | Description                                                                                                                        |
| ---------------------- | ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `simulation.completed` | `events:simulation.completed` | Published when a batch of simulations finishes. Payload includes batch_id, game_ids, league, iterations_per_game, and duration_ms. |

### Events Subscribed

None. The simulation-engine runs simulations on demand via synchronous API calls from the agent. It does not react to events autonomously.

### Storage Requirements

- **Database:** PostgreSQL (or lightweight alternative; simulation data is write-heavy but query-light).
- **Key tables:**
  - `simulation_configs` -- default and custom configs per sport (~20 rows, occasionally updated).
  - `simulation_runs` -- metadata per run (~50K-100K rows/year).
  - `simulation_results` -- output distributions per run (~50K-100K rows/year, ~5-10 KB each as JSONB).
- **Estimated row counts:** Year 1: ~50K-100K runs/results. Year 3: ~200K-300K total.
- **Growth:** ~200-500 new simulation runs/day during active seasons.
- **Indexing priorities:**
  1. `simulation_runs(game_id, parameters_hash)` -- cache lookup for identical simulations.
  2. `simulation_runs(game_id, completed_at DESC)` -- most recent simulation for a game.
  3. `simulation_runs(batch_id)` -- batch status queries.
- **Redis:** Used for simulation result cache (keyed by parameters_hash, TTL based on data freshness -- invalidated when stats.updated events indicate changed inputs). Also used for event deduplication.
