# Agent API

**Service:** agent
**Framework:** Python / FastAPI
**Base URL:** `/api/v1/agent`
**Port:** 8006

The agent is the orchestration layer and LLM-powered analyst for BookieBreaker. It runs the prediction pipeline on a
schedule, coordinates calls across all backend services, detects edges, generates natural language analysis via the
Anthropic API, and serves as the gateway for analytical queries from CLI, UI, and MCP.

> **Phase 4 (implemented):** LLM analysis (`POST /analysis`, `GET /analysis/{id}`), alerts (`GET /alerts`,
> `PUT /alerts/{alert_id}/acknowledge`), cron scheduling (`GET`/`POST /schedule`), event-triggered pipeline
> re-runs, and the LLM health dependency all landed with Phase 4. Pipeline runs are triggered on demand
> (`POST /pipeline/run`), by cron schedules (`trigger: SCHEDULED`), or by debounced `lines.updated` /
> `stats.updated` reactions (`trigger: EVENT`).

---

## Endpoints

### POST /api/v1/agent/pipeline/run

Trigger a full prediction pipeline run. The pipeline executes: stats retrieval, simulation, prediction, edge detection,
and optional paper bet placement.

**Request Body:**

```json
{
  "league": "NBA",
  "game_ids": ["g1234567-89ab-cdef-0123-456789abcdef"],
  "force_refresh": false,
  "auto_bet": true,
  "simulation_config": {
    "iterations": 10000
  }
}
```

| Field               | Type         | Required | Description                                                                            |
| ------------------- | ------------ | -------- | -------------------------------------------------------------------------------------- |
| `league`            | string       | No       | League to run pipeline for. If omitted, runs for all active leagues.                   |
| `game_ids`          | list\<UUID\> | No       | Specific games to process. If omitted, processes all upcoming games for the league(s). |
| `force_refresh`     | boolean      | No       | Force re-simulation and re-prediction even if cached. Default: false.                  |
| `auto_bet`          | boolean      | No       | Automatically place paper bets on detected edges. Default: true.                       |
| `simulation_config` | object       | No       | Override simulation parameters.                                                        |

**Response:** `202 Accepted` (pipeline runs asynchronously)

```json
{
  "data": {
    "pipeline_run_id": "pr123456-789a-bcde-f012-3456789abcde",
    "status": "running",
    "league": "NBA",
    "games_queued": 8,
    "started_at": "2026-03-30T14:22:35Z",
    "steps": {
      "simulation": "pending",
      "prediction": "pending",
      "edge_detection": "pending",
      "bet_placement": "pending"
    }
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:35Z",
    "request_id": "req-uuid"
  }
}
```

For synchronous single-game runs, the pipeline may complete within the request timeout and return `200 OK` with full
results. (The Phase 3 implementation always runs asynchronously and returns `202`; poll
`GET /pipeline/runs/{pipeline_run_id}` for progress.)

**Error:** `409 Conflict` if a pipeline run for the same league is already running (`details` includes the running
`pipeline_run_id`).

**Consumers:** CLI, UI, MCP

---

### GET /api/v1/agent/pipeline/runs/{pipeline_run_id}

Get the status and results of a pipeline run.

**Path Parameters:**

| Parameter         | Type | Description                 |
| ----------------- | ---- | --------------------------- |
| `pipeline_run_id` | UUID | The pipeline run identifier |

**Response:** `200 OK`

```json
{
  "data": {
    "pipeline_run_id": "pr123456-789a-bcde-f012-3456789abcde",
    "status": "COMPLETED",
    "trigger": "MANUAL",
    "league": "NBA",
    "params": {
      "force_refresh": false,
      "auto_bet": true
    },
    "steps": {
      "simulation": "completed",
      "prediction": "completed",
      "edge_detection": "completed",
      "bet_placement": "completed"
    },
    "games_processed": 8,
    "edges_found": 5,
    "bets_placed": 3,
    "error": null,
    "started_at": "2026-03-30T14:22:35Z",
    "finished_at": "2026-03-30T14:22:48Z"
  },
  "meta": {
    "timestamp": "2026-03-30T14:23:00Z",
    "request_id": "req-uuid"
  }
}
```

`status` is one of `QUEUED`, `RUNNING`, `COMPLETED`, `COMPLETED_WITH_ERRORS` (some games failed, others
succeeded), or `FAILED`. Per-game failures are recorded inside `steps`.

**Error:** `404 Not Found` if pipeline_run_id does not exist.

**Consumers:** CLI, UI, MCP

---

### GET /api/v1/agent/edges

List currently detected edges across all leagues.

**Query Parameters:**

| Parameter     | Type    | Required | Default | Description                                     |
| ------------- | ------- | -------- | ------- | ----------------------------------------------- |
| `league`      | string  | No       | all     | Filter by league. Comma-separated for multiple. |
| `date`        | string  | No       | today   | Filter by game date (ISO 8601 date).            |
| `min_edge`    | float   | No       | 0.0     | Minimum edge percentage to include.             |
| `market_type` | string  | No       | all     | Filter by market type.                          |
| `is_stale`    | boolean | No       | `false` | Include stale edges. Default: only fresh edges. |
| `limit`       | int     | No       | 50      | Max results per page (max 200).                 |
| `cursor`      | string  | No       | (none)  | Pagination cursor.                              |

**Response:** `200 OK`

```json
{
  "data": [
    {
      "id": "e1234567-89ab-cdef-0123-456789abcdef",
      "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
      "league": "NBA",
      "home_team": "LAL",
      "away_team": "BOS",
      "scheduled_start": "2026-03-30T22:30:00Z",
      "market_type": "TOTAL",
      "selection": "Over 220.5",
      "predicted_probability": 0.5823,
      "implied_probability": 0.5192,
      "edge_percentage": 6.31,
      "expected_value": 0.082,
      "odds_american": -108,
      "sportsbook_key": "fanduel",
      "kelly_fraction": 0.065,
      "recommended_stake": 1.63,
      "confidence": 0.62,
      "detected_at": "2026-03-30T14:22:30Z",
      "expires_at": "2026-03-30T22:30:00Z",
      "is_stale": false,
      "has_paper_bet": true,
      "paper_bet_id": "bet12345-6789-abcd-ef01-234567890abc"
    }
  ],
  "meta": {
    "timestamp": "2026-03-30T14:22:35Z",
    "request_id": "req-uuid",
    "pagination": {
      "limit": 50,
      "has_more": false,
      "next_cursor": null
    }
  }
}
```

Field notes:

- `implied_probability` is the **de-vigged** implied probability (multiplicative method by default), not the raw
  break-even probability of the quoted odds.
- `edge_percentage` is in percentage points: `(predicted_probability - implied_probability) * 100`.
- `confidence` is the edge quality score in `[0, 1]` per
  [edge-detection.md §4](../algorithms/edge-detection.md) (EV, CI width, market efficiency, line freshness,
  calibration error).
- `kelly_fraction` is the fractional-Kelly bankroll fraction (quarter Kelly, capped at 5%); `recommended_stake` is
  that fraction applied to the current bankroll in units, after simultaneous-bet exposure scaling.

**Consumers:** CLI, UI, MCP

---

### GET /api/v1/agent/edges/{edge_id}

Get detailed information about a specific edge including the prediction, line, and analysis.

**Path Parameters:**

| Parameter | Type | Description         |
| --------- | ---- | ------------------- |
| `edge_id` | UUID | The edge identifier |

**Response:** `200 OK`

```json
{
  "data": {
    "id": "e1234567-89ab-cdef-0123-456789abcdef",
    "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
    "league": "NBA",
    "game": {
      "home_team": { "id": "t1234567-...", "name": "Los Angeles Lakers", "abbreviation": "LAL" },
      "away_team": { "id": "t2345678-...", "name": "Boston Celtics", "abbreviation": "BOS" },
      "scheduled_start": "2026-03-30T22:30:00Z",
      "status": "SCHEDULED"
    },
    "market_type": "TOTAL",
    "selection": "Over 220.5",
    "predicted_probability": 0.5823,
    "simulation_probability": 0.5534,
    "implied_probability": 0.5192,
    "edge_percentage": 6.31,
    "expected_value": 0.082,
    "odds_american": -108,
    "odds_decimal": 1.926,
    "sportsbook_id": "sb345678-9abc-def0-1234-56789abcdef0",
    "sportsbook_key": "fanduel",
    "kelly_fraction": 0.065,
    "recommended_stake": 1.63,
    "confidence": 0.62,
    "detected_at": "2026-03-30T14:22:30Z",
    "expires_at": "2026-03-30T22:30:00Z",
    "is_stale": false,
    "prediction": {
      "id": "pred2345-6789-abcd-ef01-23456789abcd",
      "model_version_id": "mv123456-789a-bcde-f012-3456789abcde",
      "adjustment_magnitude": 0.0289,
      "feature_importance": {
        "pace_differential": 0.18,
        "combined_offensive_rating": 0.14,
        "rest_days_both": 0.09,
        "line_movement_direction": 0.07
      }
    },
    "betting_line": {
      "id": "d4e5f678-9012-3456-def0-123456789012",
      "sportsbook_key": "fanduel",
      "line_value": 220.5,
      "odds_american": -108,
      "timestamp": "2026-03-30T14:15:00Z"
    },
    "paper_bet": {
      "id": "bet12345-6789-abcd-ef01-234567890abc",
      "stake": 1.63,
      "result": "PENDING",
      "placed_at": "2026-03-30T14:22:35Z"
    },
    "analysis": {
      "id": "an123456-789a-bcde-f012-3456789abcde",
      "title": "Over 220.5 in LAL vs BOS",
      "content": "The simulation projects a total of 222.2 points, and the ML model adjusts upward to a 58.2% probability of the over hitting. Key factors: both teams rank in the top 10 in pace this season, LAL has gone over in 7 of their last 10 home games, and BOS plays at the 3rd highest pace in the league on the road...",
      "created_at": "2026-03-30T14:22:40Z"
    }
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:45Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if edge_id does not exist.

> `analysis` carries the newest stored LLM analysis for this edge (`{id, title, created_at}`), or `null` when
> none exists. The nested `prediction`, `betting_line`, and `paper_bet` objects are fetched live from their
> owning services and may be `null` if the owning service is unavailable.

**Consumers:** CLI, UI, MCP

---

### GET /api/v1/agent/slate

Get today's (or a given date's) games for a league with prediction summaries and active edges — the "what's on
tonight" view backing `bb slate` and the UI slate page.

**Query Parameters:**

| Parameter | Type   | Required | Default | Description                                     |
| --------- | ------ | -------- | ------- | ----------------------------------------------- |
| `league`  | string | No       | all     | Filter by league. Comma-separated for multiple. |
| `date`    | string | No       | today   | Game date (ISO 8601 date).                      |

**Response:** `200 OK`

```json
{
  "data": {
    "date": "2026-03-30",
    "games": [
      {
        "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
        "league": "NBA",
        "home_team": { "id": "t1234567-...", "name": "Los Angeles Lakers", "abbreviation": "LAL" },
        "away_team": { "id": "t2345678-...", "name": "Boston Celtics", "abbreviation": "BOS" },
        "scheduled_start": "2026-03-30T22:30:00Z",
        "status": "SCHEDULED",
        "prediction": {
          "id": "pred2345-6789-abcd-ef01-23456789abcd",
          "market_type": "MONEYLINE",
          "selection": "LAL",
          "predicted_probability": 0.5741,
          "predicted_at": "2026-03-30T14:22:20Z"
        },
        "edges": [
          {
            "id": "e1234567-89ab-cdef-0123-456789abcdef",
            "market_type": "TOTAL",
            "selection": "Over 220.5",
            "edge_percentage": 6.31,
            "sportsbook_key": "fanduel",
            "has_paper_bet": true
          }
        ]
      }
    ]
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:50Z",
    "request_id": "req-uuid"
  }
}
```

`prediction` is the latest moneyline prediction summary for the game (`null` if the game has not been through the
pipeline yet). `edges` lists fresh, unexpired edges for the game (empty list if none). Results are cached in Redis
(`agent:slate:{league}:{date}`, 5 minute TTL).

**Consumers:** CLI, UI, MCP

---

### POST /api/v1/agent/analysis

Request an LLM-generated analysis for a game, edge, or performance period.

Responses are generated by the configured LLM provider (Anthropic or Ollama per
[ADR-011](../decisions/011-local-llm-strategy.md)) and persisted to `agent.analyses`. Repeated requests for the
same subject reuse the stored analysis (cached via `agent:analysis:{type}:{scope}`, 1 hour) and return `200 OK`
instead of `201 Created`; requests with a `question` always generate fresh output. The internal daily summary
job also writes `analysis_type: "DAILY_SUMMARY"` rows, retrievable via `GET /analysis/{analysis_id}`.

**Request Body:**

```json
{
  "analysis_type": "EDGE_BREAKDOWN",
  "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
  "edge_id": "e1234567-89ab-cdef-0123-456789abcdef",
  "question": null
}
```

| Field           | Type   | Required | Description                                                         |
| --------------- | ------ | -------- | ------------------------------------------------------------------- |
| `analysis_type` | string | Yes      | Type: `GAME_PREVIEW`, `EDGE_BREAKDOWN`, `PERFORMANCE_REVIEW`.       |
| `game_id`       | UUID   | No       | Game to analyze (required for `GAME_PREVIEW` and `EDGE_BREAKDOWN`). |
| `edge_id`       | UUID   | No       | Edge to analyze (required for `EDGE_BREAKDOWN`).                    |
| `question`      | string | No       | Free-form question to answer about the game/edge/performance.       |

**Response:** `201 Created`

```json
{
  "data": {
    "id": "an123456-789a-bcde-f012-3456789abcde",
    "analysis_type": "EDGE_BREAKDOWN",
    "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
    "edge_id": "e1234567-89ab-cdef-0123-456789abcdef",
    "title": "Edge Analysis: Over 220.5 in LAL vs BOS",
    "content": "## Summary\n\nThe system identifies a 6.3% edge on the Over 220.5 in tonight's Lakers-Celtics game...\n\n## Key Factors\n\n1. **Pace**: Both teams rank in the top 10 in pace...\n2. **Recent trends**: LAL has gone over in 7 of their last 10 home games...\n3. **Injury impact**: No significant injuries affecting this total...\n\n## Risk Considerations\n\n- The line has moved up from 218.5, suggesting the market is moving in the same direction...",
    "model_used": "claude-opus-4-8",
    "input_summary": "Simulation results, team stats (LAL, BOS), line movement, injury report, last 10 game logs",
    "created_at": "2026-03-30T14:22:40Z"
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:45Z",
    "request_id": "req-uuid"
  }
}
```

**Consumers:** CLI, UI, MCP

---

### GET /api/v1/agent/analysis/{analysis_id}

Get a previously generated analysis.

**Path Parameters:**

| Parameter     | Type | Description             |
| ------------- | ---- | ----------------------- |
| `analysis_id` | UUID | The analysis identifier |

**Response:** `200 OK`

```json
{
  "data": {
    "id": "an123456-789a-bcde-f012-3456789abcde",
    "analysis_type": "EDGE_BREAKDOWN",
    "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
    "edge_id": "e1234567-89ab-cdef-0123-456789abcdef",
    "title": "Edge Analysis: Over 220.5 in LAL vs BOS",
    "content": "## Summary\n\nThe system identifies a 6.3% edge...",
    "model_used": "claude-opus-4-8",
    "input_summary": "Simulation results, team stats (LAL, BOS), line movement, injury report, last 10 game logs",
    "created_at": "2026-03-30T14:22:40Z"
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:45Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if analysis_id does not exist.

**Consumers:** CLI, UI, MCP

---

### GET /api/v1/agent/dashboard

Get aggregated dashboard data combining edges, recent performance, and pipeline status.

**Response:** `200 OK`

```json
{
  "data": {
    "active_edges": {
      "count": 5,
      "by_league": { "NBA": 3, "MLB": 2 },
      "avg_edge_pct": 4.8,
      "top_edge": {
        "id": "e1234567-89ab-cdef-0123-456789abcdef",
        "selection": "Over 220.5 LAL vs BOS",
        "edge_percentage": 6.31,
        "sportsbook_key": "fanduel"
      }
    },
    "performance_summary": {
      "today": { "bets": 3, "wins": 2, "losses": 1, "profit_units": 1.2 },
      "this_week": { "bets": 18, "wins": 11, "losses": 7, "profit_units": 5.4 },
      "all_time": { "bets": 342, "win_rate": 0.5625, "roi": 0.048, "profit_units": 19.8 }
    },
    "pipeline_status": {
      "last_run": {
        "pipeline_run_id": "pr123456-789a-bcde-f012-3456789abcde",
        "status": "completed",
        "completed_at": "2026-03-30T14:22:48Z",
        "games_processed": 8,
        "edges_found": 5,
        "bets_placed": 3
      },
      "next_scheduled_run": null
    },
    "open_bets": {
      "count": 3,
      "total_exposure_units": 4.5,
      "games_pending": 3
    }
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:50Z",
    "request_id": "req-uuid"
  }
}
```

Dashboard data is cached in Redis (`agent:dashboard:{league}`, 5 minute TTL). `next_scheduled_run` is the
earliest enabled schedule's next fire time (UTC ISO 8601), or `null` when no schedules are enabled.

**Consumers:** UI, CLI, MCP

---

### GET /api/v1/agent/schedule

Get the current pipeline schedule configuration.

**Response:** `200 OK`

```json
{
  "data": {
    "schedules": [
      {
        "id": "sched123-4567-89ab-cdef-0123456789ab",
        "league": "NBA",
        "cron_expression": "0 10,14,18 * * *",
        "timezone": "UTC",
        "description": "Run at 10 AM, 2 PM, and 6 PM daily",
        "enabled": true,
        "last_run_at": "2026-03-30T14:00:00Z",
        "next_run_at": "2026-03-30T18:00:00Z",
        "simulation_config": {
          "iterations": 10000
        },
        "auto_bet": true,
        "min_edge_threshold": 3.0
      },
      {
        "id": "sched234-5678-9abc-def0-123456789abc",
        "league": "MLB",
        "cron_expression": "0 11,16 * * *",
        "timezone": "UTC",
        "description": "Run at 11 AM and 4 PM daily",
        "enabled": true,
        "last_run_at": "2026-03-30T11:00:00Z",
        "next_run_at": "2026-03-30T16:00:00Z",
        "simulation_config": {
          "iterations": 10000
        },
        "auto_bet": true,
        "min_edge_threshold": 3.0
      }
    ]
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:50Z",
    "request_id": "req-uuid"
  }
}
```

**Consumers:** CLI, UI, MCP

---

### POST /api/v1/agent/schedule

Create or update a pipeline schedule (one per league; posting the same league updates it).

**Request Body:**

```json
{
  "league": "NBA",
  "cron_expression": "0 10,14,18 * * *",
  "timezone": "America/New_York",
  "enabled": true,
  "simulation_config": {
    "iterations": 10000
  },
  "auto_bet": true,
  "min_edge_threshold": 3.0
}
```

| Field                | Type    | Required | Description                                                      |
| -------------------- | ------- | -------- | ---------------------------------------------------------------- |
| `league`             | string  | Yes      | League for this schedule.                                        |
| `cron_expression`    | string  | Yes      | Cron expression for run timing (validated; 422 when invalid).    |
| `timezone`           | string  | No       | IANA timezone the cron expression is evaluated in. Default: UTC. |
| `enabled`            | boolean | No       | Whether the schedule is active. Default: true.                   |
| `simulation_config`  | object  | No       | Simulation config for scheduled runs.                            |
| `auto_bet`           | boolean | No       | Auto-place paper bets on edges. Default: true.                   |
| `min_edge_threshold` | float   | No       | Minimum edge percentage for auto-betting. Default: 3.0.          |

Scheduled runs fire with `trigger: "SCHEDULED"`; misfires older than the configured grace window
(`SCHEDULE_MISFIRE_GRACE_SECONDS`, default 300) roll forward without running.

**Response:** `201 Created` (new schedule) or `200 OK` (updated existing for same league)

```json
{
  "data": {
    "id": "sched123-4567-89ab-cdef-0123456789ab",
    "league": "NBA",
    "cron_expression": "0 10,14,18 * * *",
    "timezone": "America/New_York",
    "description": "Run at 10 AM, 2 PM, and 6 PM daily",
    "enabled": true,
    "next_run_at": "2026-03-30T18:00:00Z",
    "simulation_config": {
      "iterations": 10000
    },
    "auto_bet": true,
    "min_edge_threshold": 3.0
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:50Z",
    "request_id": "req-uuid"
  }
}
```

**Consumers:** CLI, UI, MCP

---

### GET /api/v1/agent/alerts

List delivered edge alerts, newest first. One alert row is persisted per `edge.detected` publish (channel
`redis`), carrying the natural-language description and the full event payload.

**Query Parameters:**

| Parameter      | Type    | Required | Default | Description                                  |
| -------------- | ------- | -------- | ------- | -------------------------------------------- |
| `priority`     | string  | No       | all     | Filter by priority: `LOW`, `MEDIUM`, `HIGH`. |
| `acknowledged` | boolean | No       | all     | Filter by acknowledgement state.             |
| `limit`        | int     | No       | 50      | Max results per page (max 200).              |
| `cursor`       | string  | No       | (none)  | Pagination cursor.                           |

**Response:** `200 OK`

```json
{
  "data": [
    {
      "id": "al123456-789a-bcde-f012-3456789abcde",
      "edge_id": "e1234567-89ab-cdef-0123-456789abcdef",
      "channel": "redis",
      "priority": "HIGH",
      "message": "6.3% edge on Over 220.5 (TOTAL, -108 at fanduel): model 58.2% vs market 51.9%.",
      "payload": {
        "event": "edge.detected",
        "edge_id": "e1234567-89ab-cdef-0123-456789abcdef",
        "description": "6.3% edge on Over 220.5 (TOTAL, -108 at fanduel): model 58.2% vs market 51.9%."
      },
      "delivered_at": "2026-03-30T14:22:31Z",
      "acknowledged_at": null
    }
  ],
  "meta": {
    "timestamp": "2026-03-30T14:22:35Z",
    "request_id": "req-uuid",
    "pagination": {
      "limit": 50,
      "has_more": false,
      "next_cursor": null
    }
  }
}
```

**Consumers:** CLI, UI, MCP

---

### PUT /api/v1/agent/alerts/{alert_id}/acknowledge

Mark an alert as acknowledged. Idempotent: re-acknowledging returns the original `acknowledged_at`.

**Path Parameters:**

| Parameter  | Type | Description          |
| ---------- | ---- | -------------------- |
| `alert_id` | UUID | The alert identifier |

**Response:** `200 OK` — the alert with `acknowledged_at` set.

**Error:** `404 Not Found` if alert_id does not exist.

**Consumers:** CLI, UI, MCP

---

### GET /api/v1/agent/health

Health check for the agent and summary of all downstream service health.

**Response:** `200 OK`

```json
{
  "data": {
    "status": "healthy",
    "service": "agent",
    "version": "1.0.0",
    "uptime_seconds": 86400,
    "dependencies": {
      "lines_service": "healthy",
      "statistics_service": "healthy",
      "simulation_engine": "healthy",
      "prediction_engine": "healthy",
      "bookie_emulator": "healthy",
      "postgres": "healthy",
      "redis": "healthy",
      "event_subscriber": "healthy"
    },
    "pipeline": {
      "last_run_status": "completed",
      "last_run_at": "2026-03-30T14:22:48Z",
      "next_scheduled_run": null
    }
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:50Z",
    "request_id": "req-uuid"
  }
}
```

The endpoint returns `200` even when dependencies are degraded (`status: "degraded"`), so container healthchecks
measure the agent itself rather than its dependencies. The LLM dependency entry is keyed by the configured
provider: `anthropic_api` when `LLM_PROVIDER=anthropic`, `ollama` when `LLM_PROVIDER=ollama`.
`pipeline.next_scheduled_run` mirrors the dashboard field.

**Consumers:** CLI, UI, MCP, infra-ops

---

## Events Published

| Event                  | Channel                       | Trigger                              |
| ---------------------- | ----------------------------- | ------------------------------------ |
| `prediction.completed` | `events:prediction.completed` | Edge detection completes for a batch |
| `edge.detected`        | `events:edge.detected`        | Actionable edge identified           |

## Events Subscribed

| Event                  | Channel                       | Purpose                                                |
| ---------------------- | ----------------------------- | ------------------------------------------------------ |
| `lines.updated`        | `events:lines.updated`        | Mark stale edges, bust caches, request an EVENT re-run |
| `stats.updated`        | `events:stats.updated`        | Request an EVENT re-run for affected leagues           |
| `game.completed`       | `events:game.completed`       | Expire the game's edges, refresh dashboard cache       |
| `simulation.completed` | `events:simulation.completed` | Monitoring and logging                                 |
| `prediction.completed` | `events:prediction.completed` | Monitoring and logging                                 |

> **Event-triggered re-runs** (Phase 4) are debounced (`RERUN_DEBOUNCE_SECONDS`, default 120) and spaced by a
> per-league cooldown (`RERUN_COOLDOWN_SECONDS`, default 600). Only leagues with an enabled schedule re-run —
> a cost guardrail so unscheduled leagues don't churn simulations on every line move. Runs fire with
> `trigger: "EVENT"`. `edge.detected` events additionally carry a natural-language `description` field
> (see [redis-schemas](../schemas/database-schemas/redis-schemas.md)).
