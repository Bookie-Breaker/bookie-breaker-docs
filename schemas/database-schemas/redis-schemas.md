# Redis Schemas

**Technology:** Redis 7
**Purpose:** Caching, pub/sub event bus, ephemeral storage
**Services using:** All services (statistics-service, simulation-engine, agent, lines-service, prediction-engine,
bookie-emulator)

---

## Overview

Redis serves as the system's cache layer and event bus. No data in Redis is authoritative -- it is always derived from
external APIs, computed on demand, or published as events. All keys have TTLs. If Redis is flushed, the system recovers
by re-fetching from source services and external APIs.

**Database allocation:** All services share a single Redis instance using key prefixes for namespace isolation. No Redis
database numbers are used (all keys in db 0).

---

## Key Patterns

### Statistics Cache (statistics-service)

The statistics-service caches data fetched from external sports APIs (nfl_data_py, nba_api, pybaseball). These caches
reduce external API calls and provide fast lookups for downstream consumers.

#### `stats:team:{league}:{team_id}`

Team statistics cache. Hash containing season-level team stats.

| Field              | Type     | Description                              |
| ------------------ | -------- | ---------------------------------------- |
| `name`             | string   | Team full name                           |
| `abbreviation`     | string   | Team short code                          |
| `wins`             | int      | Season wins                              |
| `losses`           | int      | Season losses                            |
| `offensive_rating` | float    | Points/runs scored per game              |
| `defensive_rating` | float    | Points/runs allowed per game             |
| `home_record`      | string   | Home win-loss (e.g., "8-2")              |
| `away_record`      | string   | Away win-loss                            |
| `streak`           | string   | Current streak (e.g., "W3", "L2")        |
| `last_updated`     | ISO 8601 | When this cache entry was last refreshed |

**TTL:** 6 hours
**Example key:** `stats:team:NFL:kc-chiefs`
**Set by:** statistics-service on fetch from external API
**Read by:** simulation-engine, prediction-engine, agent

---

#### `stats:player:{player_id}`

Individual player statistics cache. Hash containing current season stats and status.

| Field                | Type        | Description                           |
| -------------------- | ----------- | ------------------------------------- |
| `name`               | string      | Player full name                      |
| `team_id`            | string      | Current team identifier               |
| `position`           | string      | Position code (e.g., "QB", "PG")      |
| `status`             | string      | PlayerStatus enum value               |
| `injury_description` | string      | Injury details if applicable          |
| `games_played`       | int         | Games played this season              |
| `stats_json`         | JSON string | Sport-specific stat line (compressed) |
| `last_updated`       | ISO 8601    | Cache freshness timestamp             |

**TTL:** 6 hours
**Example key:** `stats:player:player-uuid-123`
**Set by:** statistics-service
**Read by:** prediction-engine (injury impact features), agent

---

#### `stats:game:{game_id}`

Game information cache. Hash containing schedule data and results.

| Field             | Type     | Description                             |
| ----------------- | -------- | --------------------------------------- |
| `home_team_id`    | string   | Home team identifier                    |
| `away_team_id`    | string   | Away team identifier                    |
| `league`          | string   | League enum value                       |
| `scheduled_start` | ISO 8601 | Game start time                         |
| `status`          | string   | GameStatus enum value                   |
| `home_score`      | int      | Current/final home score (if available) |
| `away_score`      | int      | Current/final away score (if available) |
| `venue_id`        | string   | Venue identifier                        |
| `last_updated`    | ISO 8601 | Cache freshness                         |

**TTL:** 15 minutes (IN_PROGRESS games), 24 hours (FINAL games), 1 hour (SCHEDULED games)
**Example key:** `stats:game:game-uuid-456`
**Set by:** statistics-service
**Read by:** bookie-emulator (grading), simulation-engine, prediction-engine, agent

---

#### `stats:features:{game_id}`

Pre-computed feature vector for a game, ready for consumption by the prediction-engine. Hash containing the derived
features that the prediction-engine needs.

| Field                   | Type     | Description                                 |
| ----------------------- | -------- | ------------------------------------------- |
| `home_rest_days`        | float    | Days since home team's last game            |
| `away_rest_days`        | float    | Days since away team's last game            |
| `home_injury_impact`    | float    | Composite injury impact score for home team |
| `away_injury_impact`    | float    | Composite injury impact score for away team |
| `travel_distance_miles` | float    | Away team travel distance                   |
| `is_dome`               | int      | 1 if dome venue, 0 otherwise                |
| `home_win_pct`          | float    | Home team season win percentage             |
| `away_win_pct`          | float    | Away team season win percentage             |
| `computed_at`           | ISO 8601 | When features were computed                 |

**TTL:** 1 hour
**Example key:** `stats:features:game-uuid-456`
**Set by:** statistics-service (computed on demand)
**Read by:** prediction-engine

---

#### `stats:schedule:{league}:{date}`

Day's schedule for a league. List of game identifiers.

**Value:** JSON array of game objects `[{"game_id": "...", "home": "KC", "away": "BUF", "start": "2026-01-18T18:30:00Z",
"status": "SCHEDULED"}, ...]`
**TTL:** 1 hour
**Example key:** `stats:schedule:NFL:2026-01-18`
**Set by:** statistics-service
**Read by:** agent (daily workflow), lines-service (polling targets), UI

---

### Simulation Cache (simulation-engine)

Simulation results are ephemeral. Only the latest run per game matters. Results are regenerated when stats change.

#### `sim:result:{game_id}:{config_hash}`

Simulation result cache. Hash containing the key outputs of a simulation run.

| Field                | Type        | Description                             |
| -------------------- | ----------- | --------------------------------------- |
| `simulation_run_id`  | string      | UUID of the simulation run              |
| `home_win_prob`      | float       | P(home wins)                            |
| `away_win_prob`      | float       | P(away wins)                            |
| `mean_home_score`    | float       | Mean home score                         |
| `mean_away_score`    | float       | Mean away score                         |
| `mean_total`         | float       | Mean combined score                     |
| `mean_margin`        | float       | Mean margin (home - away)               |
| `spread_covers_json` | JSON string | Spread cover probabilities at key lines |
| `total_overs_json`   | JSON string | Total over probabilities at key lines   |
| `iterations`         | int         | Iterations completed                    |
| `converged`          | int         | 1 if converged, 0 otherwise             |
| `completed_at`       | ISO 8601    | When the simulation finished            |

**TTL:** 2 hours
**Example key:** `sim:result:game-uuid-456:a1b2c3d4`
**Set by:** simulation-engine
**Read by:** prediction-engine, agent

---

#### `sim:distributions:{game_id}`

Full score distributions for a game. Stored as compressed JSON since distributions can be large.

**Value:** Compressed JSON string containing:

```json
{
  "simulation_run_id": "uuid",
  "home_score_dist": { "0": 0.001, "1": 0.002, "...": "..." },
  "away_score_dist": { "0": 0.001, "1": 0.002, "...": "..." },
  "margin_dist": { "-20": 0.005, "-19": 0.006, "...": "..." },
  "total_dist": { "30": 0.002, "31": 0.003, "...": "..." },
  "percentiles": {
    "margin": { "10": -8, "25": -3, "50": 2, "75": 7, "90": 13 },
    "total": { "10": 35, "25": 40, "50": 45, "75": 50, "90": 55 }
  }
}
```

**TTL:** 2 hours
**Example key:** `sim:distributions:game-uuid-456`
**Set by:** simulation-engine
**Read by:** prediction-engine (base input for ML adjustment), agent (for analysis), UI (distribution charts)

---

### Agent Cache (agent)

The agent caches dashboard summaries and analysis text to avoid re-generating expensive LLM outputs.

#### `agent:dashboard:{league}`

Pre-built dashboard data for a league. Includes today's games, open edges, recent bet results.

**Value:** JSON string containing:

```json
{
  "league": "NFL",
  "date": "2026-01-18",
  "games_today": 3,
  "open_edges": 2,
  "open_bets": 5,
  "recent_results": { "wins": 3, "losses": 2, "roi": 0.08 },
  "top_edge": { "game": "KC vs BUF", "market": "SPREAD", "edge": 0.045 },
  "generated_at": "2026-01-18T10:00:00Z"
}
```

**TTL:** 5 minutes
**Example key:** `agent:dashboard:NFL`
**Set by:** agent
**Read by:** CLI, UI, MCP server

---

#### `agent:analysis:{game_id}`

LLM-generated analysis text for a specific game.

**Value:** Plain text (markdown) containing the full game analysis narrative.
**TTL:** 1 hour
**Example key:** `agent:analysis:game-uuid-456`
**Set by:** agent (after Anthropic API call)
**Read by:** CLI, UI, MCP server

---

## Pub/Sub Channels

All inter-service events flow through Redis pub/sub channels. Events carry identifiers and metadata -- not full entity
payloads. Consumers call source service APIs to get current state after receiving an event.

### `events:lines.updated`

Published when new or changed lines are detected and persisted by the lines-service.

**Publisher:** lines-service
**Subscribers:** prediction-engine (may use line movement as feature), agent (edge re-evaluation)

**Payload schema:**

```json
{
  "event": "lines.updated",
  "timestamp": "2026-01-18T14:30:15Z",
  "league": "NFL",
  "game_ids": ["game-ext-id-1", "game-ext-id-2"],
  "market_types": ["SPREAD", "TOTAL", "MONEYLINE"],
  "sportsbooks_updated": ["draftkings", "fanduel", "pinnacle"],
  "change_count": 42,
  "source": "the_odds_api"
}
```

---

### `events:stats.updated`

Published when team/player statistics are refreshed from external APIs.

**Publisher:** statistics-service
**Subscribers:** simulation-engine (invalidate cached results), prediction-engine (refresh features)

**Payload schema:**

```json
{
  "event": "stats.updated",
  "timestamp": "2026-01-18T14:00:00Z",
  "league": "NFL",
  "update_type": "injuries",
  "team_ids": ["kc-chiefs", "buf-bills"],
  "player_ids": ["player-uuid-1", "player-uuid-2"],
  "changes": [{ "player_id": "player-uuid-1", "field": "status", "old": "ACTIVE", "new": "OUT" }]
}
```

---

### `events:game.completed`

Published when a game transitions to FINAL status with a confirmed score.

**Publisher:** statistics-service
**Subscribers:** bookie-emulator (triggers bet grading), agent (updates dashboard)

**Payload schema:**

```json
{
  "event": "game.completed",
  "timestamp": "2026-01-18T23:15:00Z",
  "game_id": "game-uuid-456",
  "game_external_id": "odds_api_abc123",
  "league": "NFL",
  "home_team": "KC",
  "away_team": "BUF",
  "home_score": 27,
  "away_score": 24,
  "total": 51,
  "margin": 3,
  "overtime": false
}
```

---

### `events:simulation.completed`

Published when a simulation run finishes for a game.

**Publisher:** simulation-engine
**Subscribers:** prediction-engine (triggers ML adjustment), agent (logs progress)

**Payload schema:**

```json
{
  "event": "simulation.completed",
  "timestamp": "2026-01-18T12:05:30Z",
  "simulation_run_id": "sim-uuid-789",
  "game_id": "game-uuid-456",
  "league": "NFL",
  "iterations": 10000,
  "converged": true,
  "duration_ms": 4200,
  "home_win_prob": 0.55,
  "mean_margin": 2.3
}
```

---

### `events:prediction.completed`

Published when the prediction-engine completes a batch of predictions.

**Publisher:** prediction-engine
**Subscribers:** agent (triggers edge detection and analysis)

**Payload schema:**

```json
{
  "event": "prediction.completed",
  "timestamp": "2026-01-18T12:06:15Z",
  "batch_id": "batch-uuid-101",
  "game_ids": ["game-uuid-456", "game-uuid-789"],
  "league": "NFL",
  "market_types": ["SPREAD", "TOTAL", "MONEYLINE"],
  "predictions_count": 18,
  "edges_found": 3
}
```

---

### `events:edge.detected`

Published when the agent identifies an actionable edge (predicted probability exceeds market-implied probability beyond
the configured threshold).

**Publisher:** agent
**Subscribers:** CLI (alert display), UI (dashboard update), MCP server (tool notification)

**Payload schema:**

```json
{
  "event": "edge.detected",
  "timestamp": "2026-01-18T12:07:00Z",
  "edge_id": "edge-uuid-202",
  "game_id": "game-uuid-456",
  "league": "NFL",
  "market_type": "SPREAD",
  "selection": "KC -3.5",
  "sportsbook": "draftkings",
  "edge_percentage": 0.042,
  "predicted_probability": 0.562,
  "implied_probability": 0.52,
  "odds_american": -110,
  "kelly_fraction": 0.08,
  "confidence": 0.85,
  "game_start": "2026-01-18T18:30:00Z",
  "priority": "HIGH"
}
```

---

## Key Naming Summary

| Pattern                              | Type                     | TTL          | Service            |
| ------------------------------------ | ------------------------ | ------------ | ------------------ |
| `stats:team:{league}:{team_id}`      | Hash                     | 6h           | statistics-service |
| `stats:player:{player_id}`           | Hash                     | 6h           | statistics-service |
| `stats:game:{game_id}`               | Hash                     | 15min/1h/24h | statistics-service |
| `stats:features:{game_id}`           | Hash                     | 1h           | statistics-service |
| `stats:schedule:{league}:{date}`     | String (JSON)            | 1h           | statistics-service |
| `sim:result:{game_id}:{config_hash}` | Hash                     | 2h           | simulation-engine  |
| `sim:distributions:{game_id}`        | String (compressed JSON) | 2h           | simulation-engine  |
| `agent:dashboard:{league}`           | String (JSON)            | 5min         | agent              |
| `agent:analysis:{game_id}`           | String (text)            | 1h           | agent              |

---

## Operational Notes

### Memory estimation

Conservative estimates for Redis memory usage:

| Category           | Keys (peak day) | Avg. size/key | Total      |
| ------------------ | --------------- | ------------- | ---------- |
| Team stats         | ~350            | ~1 KB         | ~350 KB    |
| Player stats       | ~5,000          | ~2 KB         | ~10 MB     |
| Game data          | ~100            | ~500 bytes    | ~50 KB     |
| Features           | ~50             | ~500 bytes    | ~25 KB     |
| Schedules          | ~12             | ~5 KB         | ~60 KB     |
| Simulation results | ~50             | ~2 KB         | ~100 KB    |
| Distributions      | ~50             | ~50 KB        | ~2.5 MB    |
| Agent dashboard    | ~6              | ~2 KB         | ~12 KB     |
| Agent analysis     | ~50             | ~10 KB        | ~500 KB    |
| **Total**          |                 |               | **~15 MB** |

Redis memory usage is minimal. A 256 MB Redis instance is more than sufficient.

### Eviction policy

Set `maxmemory-policy allkeys-lru`. All keys have explicit TTLs, but LRU eviction provides a safety net if memory
pressure occurs.

### Persistence

Redis persistence (RDB/AOF) is **not required**. All cached data is recoverable from source services or external APIs.
Disable persistence to maximize performance:

```text
save ""
appendonly no
```

### Pub/Sub reliability

Redis pub/sub is fire-and-forget. If a subscriber is disconnected when an event is published, the event is lost. This is
acceptable because:

1. The bookie-emulator has a fallback polling mechanism (every 30 minutes) for missed `game.completed` events.
2. The agent runs on a schedule and will pick up stale data on the next cycle.
3. No business-critical state depends solely on pub/sub delivery.

If delivery guarantees become necessary in the future, consider migrating to Redis Streams.
