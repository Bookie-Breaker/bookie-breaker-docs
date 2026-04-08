# bookie-breaker-statistics-service

## Purpose

Ingests, processes, stores, and serves historical and current sports statistics from external APIs. Serves as the
system's source of truth for all statistical data -- team stats, player stats, game results, and seasonal aggregates --
across all 6 supported leagues.

## Responsibilities

- Ingests raw statistics from external sports data APIs on a regular schedule (game results, box scores, player stats,
  team stats).
- Normalizes data into a canonical format across different source APIs and sports.
- Stores historical and current statistics with appropriate granularity (game-level, season-level, career-level).
- Computes derived statistics and aggregates (rolling averages, per-game rates, advanced metrics like
  offensive/defensive ratings, pace, efficiency).
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

| Source                    | Data                                                                          | Mechanism             |
| ------------------------- | ----------------------------------------------------------------------------- | --------------------- |
| External sports data APIs | Game results, box scores, player stats, team stats, injury reports, schedules | Scheduled API polling |
| agent                     | Requests for specific stats, matchup data, contextual data                    | API call              |
| simulation-engine         | Requests for team/player parameters needed for simulation                     | API call              |
| prediction-engine         | Requests for contextual features (injuries, rest days, etc.)                  | API call              |
| bookie-emulator           | Requests for final game scores to grade bets                                  | API call              |
| CLI / UI / MCP server     | Stats lookup requests                                                         | API call              |

## Outputs

| Destination       | Data                                                                 | Mechanism    |
| ----------------- | -------------------------------------------------------------------- | ------------ |
| simulation-engine | Team ratings, player stats, pace/efficiency metrics, matchup data    | API response |
| prediction-engine | Contextual features: injury data, rest days, travel, seasonal trends | API response |
| bookie-emulator   | Final game scores and results for bet grading                        | API response |
| agent             | Stats summaries, matchup context for LLM analysis                    | API response |
| CLI / UI          | Stats tables, player/team profiles                                   | API response |

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

---

## Requirements

### Functional Requirements

- **FR-001:** Ingest game results and box scores from sport-specific Python packages (nfl_data_py, nba_api, pybaseball,
  CFBD API, CBBD API, baseballr NCAA functions) on a scheduled cadence: every 15-30 minutes during active game windows,
  daily bulk updates otherwise.
- **FR-002:** Ingest player-level statistics (box scores, play-by-play derived stats) for all 6 leagues after each
  completed game.
- **FR-003:** Ingest injury reports and roster information at least 2-4 times daily during active seasons from available
  source APIs.
- **FR-004:** Ingest league schedules at the start of each season and refresh weekly to capture postponements,
  cancellations, and schedule changes.
- **FR-005:** Normalize all ingested data into the canonical domain model schema, standardizing team/player identifiers,
  stat names, and units across all 6 leagues and all source providers.
- **FR-006:** Maintain canonical ID mappings via `external_ids` for Teams, Players, and Games, reconciling identifiers
  across data providers (ESPN, The Odds API, nfl_data_py, nba_api, etc.).
- **FR-007:** Compute derived statistics after each game day: rolling averages (last 5/10/20 games), per-game rates,
  advanced metrics (offensive/defensive ratings, pace, efficiency), and seasonal aggregates.
- **FR-008:** Store statistics at multiple granularities: game-level (per-game box scores), season-level (aggregate
  stats), and career-level (multi-season stats).
- **FR-009:** Serve team statistics, player statistics, game results, and seasonal aggregates via REST API with
  filtering by league, team, player, date range, and stat category.
- **FR-010:** Provide final game scores for bet grading by the bookie-emulator.
- **FR-011:** Publish a `stats.updated` event to `events:stats.updated` when new statistical data is ingested, including
  league, data types updated, teams affected, and game IDs.
- **FR-012:** Publish a `game.completed` event to `events:game.completed` when a final game score is received, including
  league, game_id, teams, and final scores.
- **FR-013:** Provide contextual features for the prediction-engine: injury impact data, rest days between games, travel
  distance (computed from Venue coordinates), home/away splits, and seasonal trends.
- **FR-014:** Archive raw API responses permanently in a `raw_api_responses` TimescaleDB hypertable (source, timestamp,
  endpoint, HTTP status, response body). This data accumulates from day one for future LLM fine-tuning and training
  data. Additionally cache raw source data in Redis for 48 hours to support debugging and replay.
- **FR-015:** Recompute derived statistics within 1 minute of new game data being ingested.

### Non-Functional Requirements

- **Latency:** API responses for team/player stats must return in < 300ms. Derived stats recomputation must complete in
  < 1 minute after new game data ingestion. Game results must be available within 15-30 minutes of game completion
  (dependent on source update speed).
- **Throughput:** Handle up to 200 API requests/second from downstream consumers (simulation-engine, prediction-engine,
  bookie-emulator, agent, interfaces). Ingest up to 50 games' worth of box scores per ingestion cycle during peak game
  days.
- **Availability:** 99.5% uptime during active sports seasons. Graceful degradation: if a primary data source is
  unavailable, fall back to secondary source (e.g., ESPN endpoints for NBA if nba_api is down). Serve cached/stored data
  while sources are unavailable.
- **Storage:** Estimated 2-5 million game-level stat rows per year across all leagues. Derived/aggregate stats add ~500K
  rows/year. Injury/roster snapshots add ~200K rows/year. Total growth: ~3-6 million rows/year.

### Data Ownership

This service is the source of truth for:

- **Sport** -- the three top-level sport categories.
- **League** -- all 6 supported leagues with season metadata.
- **Team** -- all teams across all leagues with canonical identifiers and external_ids mappings.
- **Player** -- all players with current team, position, status, injury information, and external_ids mappings.
- **Venue** -- all stadiums/arenas with geographic coordinates, surface type, dome status, and capacity.
- **Game** -- all scheduled, in-progress, and completed games with scores and external_ids.
- **GameResult** -- detailed final results including period scores, overtime status, and completion timestamps.

### APIs Exposed

| Method + Path                                       | Description                                      | Key Query Parameters                            | Consumers                                                 |
| --------------------------------------------------- | ------------------------------------------------ | ----------------------------------------------- | --------------------------------------------------------- |
| `GET /api/v1/teams`                                 | List teams                                       | `league`, `conference`, `division`, `active`    | simulation-engine, prediction-engine, agent, CLI, UI, MCP |
| `GET /api/v1/teams/{team_id}/stats`                 | Get team statistics                              | `season`, `stat_type`, `rolling_window`         | simulation-engine, prediction-engine, agent               |
| `GET /api/v1/players`                               | List players                                     | `team_id`, `league`, `position`, `status`       | prediction-engine, agent, CLI, UI, MCP                    |
| `GET /api/v1/players/{player_id}/stats`             | Get player statistics                            | `season`, `stat_type`, `game_log`               | simulation-engine, prediction-engine, agent               |
| `GET /api/v1/games`                                 | List games                                       | `league`, `date`, `team_id`, `status`, `season` | all services and interfaces                               |
| `GET /api/v1/games/{game_id}`                       | Get game details and result                      | --                                              | bookie-emulator (grading), agent, CLI, UI                 |
| `GET /api/v1/games/{game_id}/result`                | Get detailed game result                         | --                                              | bookie-emulator (bet grading)                             |
| `GET /api/v1/games/{game_id}/boxscore`              | Get full box score                               | --                                              | agent (LLM context), CLI, UI                              |
| `GET /api/v1/injuries`                              | Get current injury reports                       | `league`, `team_id`, `status`                   | prediction-engine (contextual features), agent            |
| `GET /api/v1/matchup/{home_team_id}/{away_team_id}` | Get matchup context (head-to-head, rest, travel) | `season`, `league`                              | simulation-engine, prediction-engine, agent               |
| `GET /api/v1/leagues`                               | List leagues with season metadata                | --                                              | agent (scheduling), all services                          |
| `GET /api/v1/venues/{venue_id}`                     | Get venue details                                | --                                              | prediction-engine (contextual features)                   |
| `GET /api/v1/schedule`                              | Get upcoming schedule                            | `league`, `date_range`, `team_id`               | agent (pipeline scheduling)                               |

### APIs Consumed

| Service                   | Endpoint/Package          | Purpose                                                  |
| ------------------------- | ------------------------- | -------------------------------------------------------- |
| nfl_data_py (external)    | Python package            | NFL game results, play-by-play, player stats, rosters    |
| nba_api (external)        | Python package            | NBA box scores, player stats, team stats, schedules      |
| pybaseball (external)     | Python package (Statcast) | MLB game results, player stats, pitch-level data         |
| CFBD API (external)       | REST API                  | NCAA Football game results, stats, schedules             |
| CBBD API (external)       | REST API                  | NCAA Basketball game results, stats                      |
| baseballr NCAA (external) | R package via rpy2        | NCAA Baseball game results and stats from stats.ncaa.org |
| ESPN endpoints (external) | Undocumented REST         | Fallback for injury reports, schedules, scores           |

### Events Published

| Event            | Channel                 | Description                                                                                                                                                           |
| ---------------- | ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `stats.updated`  | `events:stats.updated`  | Published when new statistical data is ingested. Payload includes league, data_types (e.g., game_results, player_stats, injury_report), teams_affected, and game_ids. |
| `game.completed` | `events:game.completed` | Published when a final game score is received. Payload includes league, game_id, home_team, away_team, home_score, away_score, and status.                            |

### Events Subscribed

None. The statistics-service is a data producer only; it ingests from external sources on its own schedule.

### Storage Requirements

- **Database:** PostgreSQL (primary persistent store).
- **Key tables:**
  - `sports` -- sport categories (3 rows, static).
  - `leagues` -- league definitions with season metadata (6 rows, near-static).
  - `teams` -- team registry with external_ids (~400 rows across all leagues, updated seasonally).
  - `players` -- player registry with status and external_ids (~15,000 active players across all leagues, updated
    daily).
  - `venues` -- stadium/arena details (~300 rows, near-static).
  - `games` -- game schedule and results (~15,000 games/year across all leagues).
  - `game_results` -- detailed final results (~10,000 completed games/year).
  - `game_stats` -- game-level box score stats (~2-5M rows/year, varies by sport granularity).
  - `player_stats` -- player-level per-game stats (~2-5M rows/year).
  - `team_season_stats` -- aggregated team season stats (~2,400 rows/year, recomputed).
  - `injuries` -- current and historical injury reports (~200K snapshots/year).
  - `ingestion_log` -- tracks each ingestion cycle (~100K rows/year).
- **Estimated row counts:** Year 1: ~5-10M rows total. Year 3: ~20-30M rows.
- **Growth:** ~20,000-50,000 new rows/day during peak multi-sport overlap.
- **Indexing priorities:**
  1. `games(league_id, scheduled_start, status)` -- primary query pattern for upcoming/recent games.
  2. `games(external_ids)` -- GIN index for cross-provider reconciliation.
  3. `player_stats(player_id, season, game_id)` -- player stat lookups.
  4. `game_stats(game_id, team_id)` -- box score lookups.
  5. `injuries(team_id, status)` -- current injury report lookups.
  6. `teams(league_id, abbreviation)` -- team resolution.
- **Redis:** Used for raw source data cache (48h TTL), frequently queried team/player stats cache (15m TTL), and event
  deduplication (1h TTL).
