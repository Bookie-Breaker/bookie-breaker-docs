# Statistics Service API

**Service:** statistics-service
**Framework:** Go / Echo
**Base URL:** `/api/v1/stats`
**Port:** 8002

The statistics-service ingests, normalizes, and serves sports statistics from external APIs. It acts as a cache and enrichment layer -- fetching from external sources, caching in Redis with generous TTLs, and computing derived features in-memory. External APIs are the source of truth; this service does not maintain a persistent statistics database.

---

## Endpoints

### GET /api/v1/stats/teams

List teams with optional filters.

**Query Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `league` | string | No | all | Filter by league (e.g., `NBA`, `NFL`). Comma-separated for multiple. |
| `season` | int | No | current | Season year. |
| `conference` | string | No | (none) | Filter by conference. |
| `division` | string | No | (none) | Filter by division. |
| `active` | boolean | No | `true` | Filter by active status. |
| `limit` | int | No | 50 | Max results per page (max 200). |
| `cursor` | string | No | (none) | Pagination cursor. |

**Response:** `200 OK`

```json
{
  "data": [
    {
      "id": "t1234567-89ab-cdef-0123-456789abcdef",
      "league_id": "l1234567-89ab-cdef-0123-456789abcdef",
      "league": "NBA",
      "name": "Los Angeles Lakers",
      "abbreviation": "LAL",
      "location": "Los Angeles",
      "mascot": "Lakers",
      "conference": "Western",
      "division": "Pacific",
      "venue_id": "v1234567-89ab-cdef-0123-456789abcdef",
      "active": true,
      "external_ids": {
        "espn": "13",
        "nba_api": "1610612747",
        "odds_api": "los-angeles-lakers"
      }
    }
  ],
  "meta": {
    "timestamp": "2026-03-30T14:22:05Z",
    "request_id": "req-uuid",
    "pagination": {
      "limit": 50,
      "has_more": true,
      "next_cursor": "eyJpZCI6..."
    }
  }
}
```

**Consumers:** simulation-engine, prediction-engine, agent, CLI, UI, MCP

---

### GET /api/v1/stats/teams/{team_id}

Get team details with current season summary stats.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `team_id` | UUID | The team identifier |

**Response:** `200 OK`

```json
{
  "data": {
    "id": "t1234567-89ab-cdef-0123-456789abcdef",
    "league": "NBA",
    "name": "Los Angeles Lakers",
    "abbreviation": "LAL",
    "location": "Los Angeles",
    "mascot": "Lakers",
    "conference": "Western",
    "division": "Pacific",
    "venue": {
      "id": "v1234567-89ab-cdef-0123-456789abcdef",
      "name": "Crypto.com Arena",
      "city": "Los Angeles",
      "state": "CA"
    },
    "season_summary": {
      "season": 2026,
      "wins": 42,
      "losses": 28,
      "win_pct": 0.6,
      "points_per_game": 114.2,
      "points_allowed_per_game": 110.5,
      "offensive_rating": 115.3,
      "defensive_rating": 111.2
    }
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:05Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if team_id does not exist.

**Consumers:** simulation-engine, prediction-engine, agent, CLI, UI, MCP

---

### GET /api/v1/stats/teams/{team_id}/stats

Get detailed team statistics with support for rolling windows and stat types.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `team_id` | UUID | The team identifier |

**Query Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `season` | int | No | current | Season year. |
| `stat_type` | string | No | `all` | Stat category: `offensive`, `defensive`, `overall`, `advanced`. |
| `rolling_window` | int | No | (none) | Rolling average over last N games (e.g., `5`, `10`, `20`). |

**Response:** `200 OK`

```json
{
  "data": {
    "team_id": "t1234567-89ab-cdef-0123-456789abcdef",
    "team_abbreviation": "LAL",
    "season": 2026,
    "games_played": 70,
    "stats": {
      "offensive": {
        "points_per_game": 114.2,
        "field_goal_pct": 0.472,
        "three_point_pct": 0.365,
        "free_throw_pct": 0.782,
        "rebounds_per_game": 44.8,
        "assists_per_game": 26.1,
        "turnovers_per_game": 13.5,
        "offensive_rating": 115.3,
        "pace": 100.2,
        "effective_fg_pct": 0.541
      },
      "defensive": {
        "points_allowed_per_game": 110.5,
        "opponent_fg_pct": 0.458,
        "opponent_three_point_pct": 0.348,
        "steals_per_game": 7.8,
        "blocks_per_game": 5.2,
        "defensive_rating": 111.2
      },
      "advanced": {
        "net_rating": 4.1,
        "true_shooting_pct": 0.582,
        "turnover_pct": 0.128,
        "offensive_rebound_pct": 0.285
      }
    },
    "home_away_splits": {
      "home": {
        "wins": 25,
        "losses": 10,
        "points_per_game": 117.1,
        "points_allowed_per_game": 108.2
      },
      "away": {
        "wins": 17,
        "losses": 18,
        "points_per_game": 111.3,
        "points_allowed_per_game": 112.8
      }
    }
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:05Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if team_id does not exist.

**Consumers:** simulation-engine, prediction-engine, agent

---

### GET /api/v1/stats/players/{player_id}

Get player details with current season summary stats.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `player_id` | UUID | The player identifier |

**Query Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `season` | int | No | current | Season year. |
| `stat_type` | string | No | `all` | Stat category filter. |
| `game_log` | boolean | No | `false` | Include recent game log entries. |

**Response:** `200 OK`

```json
{
  "data": {
    "id": "p1234567-89ab-cdef-0123-456789abcdef",
    "team_id": "t1234567-89ab-cdef-0123-456789abcdef",
    "team_abbreviation": "LAL",
    "first_name": "LeBron",
    "last_name": "James",
    "position": "SF",
    "jersey_number": 23,
    "status": "ACTIVE",
    "injury_description": null,
    "experience_years": 23,
    "season_stats": {
      "season": 2026,
      "games_played": 62,
      "minutes_per_game": 34.5,
      "points_per_game": 25.1,
      "rebounds_per_game": 7.8,
      "assists_per_game": 8.2,
      "steals_per_game": 1.2,
      "blocks_per_game": 0.6,
      "field_goal_pct": 0.512,
      "three_point_pct": 0.378,
      "free_throw_pct": 0.745
    }
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:05Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if player_id does not exist.

**Consumers:** simulation-engine, prediction-engine, agent, CLI, UI, MCP

---

### GET /api/v1/stats/games

List games with optional filters.

**Query Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `league` | string | No | all | Filter by league. Comma-separated for multiple. |
| `date_from` | string | No | (none) | Start date (ISO 8601 date). |
| `date_to` | string | No | (none) | End date (ISO 8601 date). |
| `team` | string | No | (none) | Filter by team UUID or abbreviation. |
| `status` | string | No | (none) | Filter by game status (e.g., `SCHEDULED`, `FINAL`). Comma-separated for multiple. |
| `season` | int | No | current | Season year. |
| `limit` | int | No | 50 | Max results per page (max 200). |
| `cursor` | string | No | (none) | Pagination cursor. |

**Response:** `200 OK`

```json
{
  "data": [
    {
      "id": "g1234567-89ab-cdef-0123-456789abcdef",
      "league": "NBA",
      "home_team": {
        "id": "t1234567-89ab-cdef-0123-456789abcdef",
        "name": "Los Angeles Lakers",
        "abbreviation": "LAL"
      },
      "away_team": {
        "id": "t2345678-9abc-def0-1234-56789abcdef0",
        "name": "Boston Celtics",
        "abbreviation": "BOS"
      },
      "venue": {
        "id": "v1234567-89ab-cdef-0123-456789abcdef",
        "name": "Crypto.com Arena"
      },
      "scheduled_start": "2026-03-30T22:30:00Z",
      "status": "SCHEDULED",
      "season": 2026,
      "season_type": "REGULAR",
      "home_score": null,
      "away_score": null
    }
  ],
  "meta": {
    "timestamp": "2026-03-30T14:22:05Z",
    "request_id": "req-uuid",
    "pagination": {
      "limit": 50,
      "has_more": false
    }
  }
}
```

**Consumers:** all services and interfaces

---

### GET /api/v1/stats/games/{game_id}

Get game details with results if completed.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `game_id` | UUID | The game identifier |

**Response:** `200 OK`

```json
{
  "data": {
    "id": "g1234567-89ab-cdef-0123-456789abcdef",
    "league": "NBA",
    "home_team": {
      "id": "t1234567-89ab-cdef-0123-456789abcdef",
      "name": "Los Angeles Lakers",
      "abbreviation": "LAL"
    },
    "away_team": {
      "id": "t2345678-9abc-def0-1234-56789abcdef0",
      "name": "Boston Celtics",
      "abbreviation": "BOS"
    },
    "venue": {
      "id": "v1234567-89ab-cdef-0123-456789abcdef",
      "name": "Crypto.com Arena"
    },
    "scheduled_start": "2026-03-29T22:30:00Z",
    "status": "FINAL",
    "season": 2026,
    "season_type": "REGULAR",
    "home_score": 112,
    "away_score": 108,
    "result": {
      "id": "r1234567-89ab-cdef-0123-456789abcdef",
      "home_score": 112,
      "away_score": 108,
      "total_score": 220,
      "margin": 4,
      "overtime": false,
      "completed_at": "2026-03-30T01:15:00Z",
      "period_scores": [
        { "period": 1, "home": 28, "away": 30 },
        { "period": 2, "home": 32, "away": 25 },
        { "period": 3, "home": 24, "away": 28 },
        { "period": 4, "home": 28, "away": 25 }
      ]
    }
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:05Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if game_id does not exist.

**Consumers:** bookie-emulator (grading), agent, CLI, UI

---

### GET /api/v1/stats/games/{game_id}/box-score

Get detailed box score statistics for a game.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `game_id` | UUID | The game identifier |

**Response:** `200 OK`

```json
{
  "data": {
    "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
    "status": "FINAL",
    "home_team": {
      "id": "t1234567-89ab-cdef-0123-456789abcdef",
      "abbreviation": "LAL",
      "score": 112,
      "team_stats": {
        "field_goals_made": 42,
        "field_goals_attempted": 88,
        "three_pointers_made": 12,
        "three_pointers_attempted": 34,
        "free_throws_made": 16,
        "free_throws_attempted": 20,
        "rebounds": 46,
        "assists": 28,
        "turnovers": 12,
        "steals": 8,
        "blocks": 5
      },
      "players": [
        {
          "player_id": "p1234567-89ab-cdef-0123-456789abcdef",
          "name": "LeBron James",
          "position": "SF",
          "minutes": 36,
          "points": 28,
          "rebounds": 9,
          "assists": 10,
          "steals": 2,
          "blocks": 1,
          "field_goals": "10-18",
          "three_pointers": "3-7",
          "free_throws": "5-6"
        }
      ]
    },
    "away_team": {
      "id": "t2345678-9abc-def0-1234-56789abcdef0",
      "abbreviation": "BOS",
      "score": 108,
      "team_stats": { },
      "players": [ ]
    }
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:05Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if game_id does not exist or game has not started.

**Consumers:** agent (LLM context), CLI, UI

---

### GET /api/v1/stats/features/{game_id}

Get a pre-computed feature vector for a specific game. This endpoint provides structured input data for the simulation-engine and prediction-engine.

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `game_id` | UUID | The game identifier |

**Response:** `200 OK`

```json
{
  "data": {
    "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
    "league": "NBA",
    "computed_at": "2026-03-30T14:20:00Z",
    "home_team": {
      "team_id": "t1234567-89ab-cdef-0123-456789abcdef",
      "abbreviation": "LAL",
      "offensive_rating": 115.3,
      "defensive_rating": 111.2,
      "pace": 100.2,
      "net_rating": 4.1,
      "rest_days": 2,
      "home_record": "25-10",
      "last_10": "7-3",
      "injuries": [
        {
          "player_id": "p9876543-21ab-cdef-0123-456789abcdef",
          "name": "Anthony Davis",
          "position": "PF",
          "status": "INJURED",
          "injury_description": "Knee soreness",
          "impact_score": -2.5
        }
      ]
    },
    "away_team": {
      "team_id": "t2345678-9abc-def0-1234-56789abcdef0",
      "abbreviation": "BOS",
      "offensive_rating": 118.1,
      "defensive_rating": 108.5,
      "pace": 98.5,
      "net_rating": 9.6,
      "rest_days": 1,
      "away_record": "22-13",
      "last_10": "8-2",
      "injuries": []
    },
    "matchup_context": {
      "head_to_head_season": { "home_wins": 1, "away_wins": 1 },
      "travel_distance_miles": 2451,
      "venue_is_dome": true
    }
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:05Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if game_id does not exist.

**Consumers:** simulation-engine, prediction-engine

---

### GET /api/v1/stats/leagues

List all supported leagues with season metadata.

**Response:** `200 OK`

```json
{
  "data": [
    {
      "id": "l1234567-89ab-cdef-0123-456789abcdef",
      "sport": "BASKETBALL",
      "name": "NBA",
      "display_name": "National Basketball Association",
      "abbreviation": "NBA",
      "season_type": "REGULAR",
      "current_season": 2026,
      "regular_season_games": 82,
      "season_start_month": 10,
      "season_end_month": 4,
      "has_playoffs": true
    },
    {
      "id": "l2345678-9abc-def0-1234-56789abcdef0",
      "sport": "FOOTBALL",
      "name": "NFL",
      "display_name": "National Football League",
      "abbreviation": "NFL",
      "season_type": "OFFSEASON",
      "current_season": 2025,
      "regular_season_games": 17,
      "season_start_month": 9,
      "season_end_month": 1,
      "has_playoffs": true
    }
  ],
  "meta": {
    "timestamp": "2026-03-30T14:22:05Z",
    "request_id": "req-uuid"
  }
}
```

**Consumers:** agent (scheduling), all services

---

### POST /api/v1/stats/cache/refresh

Manually trigger a cache refresh for specific leagues or data types.

**Request Body:**

```json
{
  "leagues": ["NBA"],
  "data_types": ["game_results", "player_stats", "injury_report"]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `leagues` | list\<string\> | No | Leagues to refresh. Default: all active leagues. |
| `data_types` | list\<string\> | No | Data types to refresh: `game_results`, `player_stats`, `team_stats`, `injury_report`, `schedule`. Default: all. |

**Response:** `202 Accepted`

```json
{
  "data": {
    "refresh_id": "ref12345-6789-abcd-ef01-234567890abc",
    "status": "started",
    "leagues": ["NBA"],
    "data_types": ["game_results", "player_stats", "injury_report"],
    "started_at": "2026-03-30T14:22:05Z"
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:05Z",
    "request_id": "req-uuid"
  }
}
```

**Consumers:** agent (debugging), CLI (manual operations)

---

### GET /api/v1/stats/health

Health check for the statistics-service.

**Response:** `200 OK`

```json
{
  "data": {
    "status": "healthy",
    "service": "statistics-service",
    "version": "1.0.0",
    "uptime_seconds": 86400,
    "dependencies": {
      "redis": "healthy",
      "nba_api": "healthy",
      "nfl_data_py": "healthy",
      "pybaseball": "healthy",
      "cfbd_api": "healthy"
    },
    "cache_stats": {
      "hit_rate": 0.87,
      "total_keys": 4521,
      "memory_used_mb": 128
    }
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:05Z",
    "request_id": "req-uuid"
  }
}
```

**Consumers:** agent, infra-ops

---

## Events Published

| Event | Channel | Trigger |
|-------|---------|---------|
| `stats.updated` | `events:stats.updated` | New statistical data ingested |
| `game.completed` | `events:game.completed` | Final game score received |

## Events Subscribed

None.
