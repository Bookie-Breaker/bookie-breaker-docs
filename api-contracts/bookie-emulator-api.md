# Bookie Emulator API

**Service:** bookie-emulator
**Framework:** Python / FastAPI
**Base URL:** `/api/v1/emulator`
**Port:** 8005

The bookie-emulator is a paper trading system that places virtual bets when edges are detected, tracks them through game completion, grades results, and computes performance metrics (ROI, CLV, calibration, win rate).

---

## Endpoints

### POST /api/v1/emulator/bets

Place a new paper bet.

**Headers:**

| Header              | Required | Description                                           |
| ------------------- | -------- | ----------------------------------------------------- |
| `X-Idempotency-Key` | Yes      | UUID to prevent duplicate bet placement from retries. |

**Request Body:**

```json
{
  "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
  "edge_id": "e1234567-89ab-cdef-0123-456789abcdef",
  "market_type": "SPREAD",
  "selection": "LAL -3.5",
  "side": "HOME",
  "predicted_probability": 0.5312,
  "edge_percentage": 4.2,
  "stake": 1.5,
  "reasoning": "Strong home rest advantage with key away injuries. Line movement supports value on home spread."
}
```

| Field                   | Type   | Required | Description                                            |
| ----------------------- | ------ | -------- | ------------------------------------------------------ |
| `game_id`               | UUID   | Yes      | The game being bet on.                                 |
| `edge_id`               | UUID   | Yes      | The edge that triggered this bet.                      |
| `market_type`           | string | Yes      | Market type enum value.                                |
| `selection`             | string | Yes      | Human-readable selection (e.g., "LAL -3.5").           |
| `side`                  | string | Yes      | Bet side enum value (`HOME`, `AWAY`, `OVER`, `UNDER`). |
| `predicted_probability` | float  | Yes      | System's predicted probability.                        |
| `edge_percentage`       | float  | Yes      | Edge size at time of placement.                        |
| `stake`                 | float  | Yes      | Stake in units.                                        |
| `reasoning`             | string | No       | Brief explanation of why this bet was placed.          |

**Response:** `201 Created`

```json
{
  "data": {
    "id": "bet12345-6789-abcd-ef01-234567890abc",
    "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
    "edge_id": "e1234567-89ab-cdef-0123-456789abcdef",
    "sportsbook_id": "sb123456-789a-bcde-f012-3456789abcde",
    "sportsbook_key": "draftkings",
    "market_type": "SPREAD",
    "selection": "LAL -3.5",
    "side": "HOME",
    "line_value": -3.5,
    "odds_american": -110,
    "odds_decimal": 1.909,
    "stake": 1.5,
    "stake_dollars": 150.0,
    "predicted_probability": 0.5312,
    "edge_percentage": 4.2,
    "reasoning": "Strong home rest advantage with key away injuries. Line movement supports value on home spread.",
    "result": "PENDING",
    "profit_loss": null,
    "placed_at": "2026-03-30T14:22:35Z",
    "graded_at": null
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:35Z",
    "request_id": "req-uuid"
  }
}
```

The service captures current odds from lines-service at placement time. If an identical `X-Idempotency-Key` is received, the existing bet is returned with `200 OK`.

**Error:** `400 Bad Request` if required fields are missing or invalid. `422 Unprocessable Entity` if the game has already started or the stake exceeds bankroll limits. `502 Bad Gateway` if lines-service is unavailable for odds capture.

**Consumers:** agent

---

### GET /api/v1/emulator/bets

List paper bets (bet ledger) with filtering.

**Query Parameters:**

| Parameter     | Type   | Required | Default | Description                                                |
| ------------- | ------ | -------- | ------- | ---------------------------------------------------------- |
| `league`      | string | No       | all     | Filter by league.                                          |
| `market_type` | string | No       | all     | Filter by market type.                                     |
| `result`      | string | No       | all     | Filter by result (`WIN`, `LOSS`, `PUSH`, `PENDING`).       |
| `date_from`   | string | No       | (none)  | Start date (ISO 8601).                                     |
| `date_to`     | string | No       | (none)  | End date (ISO 8601).                                       |
| `min_edge`    | float  | No       | (none)  | Minimum edge percentage.                                   |
| `status`      | string | No       | all     | `open` for pending bets, `graded` for completed, or `all`. |
| `limit`       | int    | No       | 50      | Max results per page (max 200).                            |
| `cursor`      | string | No       | (none)  | Pagination cursor.                                         |

**Response:** `200 OK`

```json
{
  "data": [
    {
      "id": "bet12345-6789-abcd-ef01-234567890abc",
      "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
      "sportsbook_key": "draftkings",
      "market_type": "SPREAD",
      "selection": "LAL -3.5",
      "side": "HOME",
      "line_value": -3.5,
      "odds_american": -110,
      "odds_decimal": 1.909,
      "stake": 1.5,
      "stake_dollars": 150.0,
      "predicted_probability": 0.5312,
      "edge_percentage": 4.2,
      "result": "WIN",
      "profit_loss": 1.364,
      "profit_loss_dollars": 136.36,
      "clv": 0.015,
      "placed_at": "2026-03-30T14:22:35Z",
      "graded_at": "2026-03-31T01:30:00Z"
    }
  ],
  "meta": {
    "timestamp": "2026-03-30T14:22:35Z",
    "request_id": "req-uuid",
    "pagination": {
      "limit": 50,
      "has_more": false
    }
  }
}
```

**Consumers:** agent, CLI, UI, MCP

---

### GET /api/v1/emulator/bets/{bet_id}

Get detailed information about a specific bet including grade details.

**Path Parameters:**

| Parameter | Type | Description              |
| --------- | ---- | ------------------------ |
| `bet_id`  | UUID | The paper bet identifier |

**Response:** `200 OK`

```json
{
  "data": {
    "id": "bet12345-6789-abcd-ef01-234567890abc",
    "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
    "edge_id": "e1234567-89ab-cdef-0123-456789abcdef",
    "sportsbook_id": "sb123456-789a-bcde-f012-3456789abcde",
    "sportsbook_key": "draftkings",
    "market_type": "SPREAD",
    "selection": "LAL -3.5",
    "side": "HOME",
    "line_value": -3.5,
    "odds_american": -110,
    "odds_decimal": 1.909,
    "stake": 1.5,
    "stake_dollars": 150.0,
    "predicted_probability": 0.5312,
    "edge_percentage": 4.2,
    "reasoning": "Strong home rest advantage with key away injuries.",
    "result": "WIN",
    "profit_loss": 1.364,
    "profit_loss_dollars": 136.36,
    "closing_line_value": -4.0,
    "closing_odds_american": -110,
    "clv": 0.015,
    "placed_at": "2026-03-30T14:22:35Z",
    "graded_at": "2026-03-31T01:30:00Z",
    "grade": {
      "id": "bg123456-789a-bcde-f012-3456789abcde",
      "game_result_id": "r1234567-89ab-cdef-0123-456789abcdef",
      "actual_home_score": 112,
      "actual_away_score": 108,
      "actual_margin": 4,
      "actual_total": 220,
      "result": "WIN",
      "result_description": "LAL won by 4, covering -3.5",
      "graded_at": "2026-03-31T01:30:00Z"
    }
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:35Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if bet_id does not exist.

**Consumers:** agent, CLI, UI, MCP

---

### POST /api/v1/emulator/bets/{bet_id}/grade

Manually grade a bet. Used when automatic grading (via `game.completed` event) has not fired.

**Path Parameters:**

| Parameter | Type | Description              |
| --------- | ---- | ------------------------ |
| `bet_id`  | UUID | The paper bet identifier |

**Request Body:**

```json
{
  "force": false
}
```

| Field   | Type    | Required | Description                                               |
| ------- | ------- | -------- | --------------------------------------------------------- |
| `force` | boolean | No       | If true, re-grade even if already graded. Default: false. |

**Response:** `200 OK`

Returns the same structure as `GET /api/v1/emulator/bets/{bet_id}` with the grade populated.

**Error:** `404 Not Found` if bet_id does not exist. `422 Unprocessable Entity` if the game has not completed yet. `409 Conflict` if bet is already graded and `force` is false.

**Consumers:** agent, CLI

---

### GET /api/v1/emulator/performance

Get aggregate performance metrics.

**Query Parameters:**

| Parameter     | Type   | Required | Default    | Description                                            |
| ------------- | ------ | -------- | ---------- | ------------------------------------------------------ |
| `league`      | string | No       | all        | Filter by league.                                      |
| `market_type` | string | No       | all        | Filter by market type.                                 |
| `date_from`   | string | No       | (none)     | Start date (ISO 8601).                                 |
| `date_to`     | string | No       | (none)     | End date (ISO 8601).                                   |
| `window`      | string | No       | `all_time` | Time window: `daily`, `weekly`, `monthly`, `all_time`. |

**Response:** `200 OK`

```json
{
  "data": {
    "period": {
      "from": "2026-01-01T00:00:00Z",
      "to": "2026-03-30T23:59:59Z",
      "window": "all_time"
    },
    "total_bets": 342,
    "total_wins": 189,
    "total_losses": 147,
    "total_pushes": 6,
    "win_rate": 0.5625,
    "roi": 0.048,
    "total_wagered_units": 412.5,
    "total_profit_units": 19.8,
    "total_wagered_dollars": 41250.0,
    "total_profit_dollars": 1980.0,
    "avg_odds_american": -108,
    "avg_edge_percentage": 3.8,
    "avg_clv": 0.012,
    "longest_win_streak": 8,
    "longest_loss_streak": 5,
    "brier_score": 0.228,
    "calibration_error": 0.022
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:35Z",
    "request_id": "req-uuid"
  }
}
```

**Consumers:** agent, CLI, UI, MCP

---

### GET /api/v1/emulator/performance/breakdown

Get performance broken down by league, bet type, or time period.

**Query Parameters:**

| Parameter   | Type   | Required | Default  | Description                                                         |
| ----------- | ------ | -------- | -------- | ------------------------------------------------------------------- |
| `group_by`  | string | No       | `league` | Grouping dimension: `league`, `market_type`, `sportsbook`, `month`. |
| `date_from` | string | No       | (none)   | Start date (ISO 8601).                                              |
| `date_to`   | string | No       | (none)   | End date (ISO 8601).                                                |

**Response:** `200 OK`

```json
{
  "data": {
    "group_by": "league",
    "breakdowns": [
      {
        "group": "NBA",
        "total_bets": 180,
        "wins": 102,
        "losses": 75,
        "pushes": 3,
        "win_rate": 0.576,
        "roi": 0.062,
        "total_profit_units": 14.2,
        "avg_clv": 0.015,
        "avg_edge_percentage": 4.1
      },
      {
        "group": "MLB",
        "total_bets": 95,
        "wins": 52,
        "losses": 41,
        "pushes": 2,
        "win_rate": 0.559,
        "roi": 0.035,
        "total_profit_units": 4.1,
        "avg_clv": 0.009,
        "avg_edge_percentage": 3.5
      }
    ]
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:35Z",
    "request_id": "req-uuid"
  }
}
```

**Consumers:** agent, CLI, UI, MCP

---

### GET /api/v1/emulator/bankroll

Get current bankroll state.

**Response:** `200 OK`

```json
{
  "data": {
    "bankroll_units": 119.8,
    "bankroll_dollars": 11980.0,
    "unit_size_dollars": 100.0,
    "starting_bankroll_units": 100.0,
    "total_profit_units": 19.8,
    "open_bets_count": 3,
    "open_bets_exposure_units": 4.5,
    "config": {
      "max_bet_units": 3.0,
      "max_daily_exposure_units": 10.0,
      "kelly_fraction": 0.25,
      "kelly_enabled": true
    },
    "snapshot_at": "2026-03-30T14:00:00Z"
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:35Z",
    "request_id": "req-uuid"
  }
}
```

**Consumers:** agent, CLI, UI, MCP

---

### GET /api/v1/emulator/bankroll/history

Get bankroll snapshots over time for charting.

**Query Parameters:**

| Parameter   | Type   | Required | Default     | Description                                      |
| ----------- | ------ | -------- | ----------- | ------------------------------------------------ |
| `date_from` | string | No       | 30 days ago | Start date (ISO 8601).                           |
| `date_to`   | string | No       | now         | End date (ISO 8601).                             |
| `interval`  | string | No       | `daily`     | Snapshot interval: `per_bet`, `daily`, `weekly`. |

**Response:** `200 OK`

```json
{
  "data": {
    "interval": "daily",
    "snapshots": [
      {
        "timestamp": "2026-03-01T23:59:59Z",
        "bankroll_units": 103.2,
        "bankroll_dollars": 10320.0,
        "total_bets": 15,
        "total_wins": 9,
        "total_losses": 6,
        "win_rate": 0.6,
        "roi": 0.032,
        "units_won": 3.2,
        "avg_clv": 0.018
      },
      {
        "timestamp": "2026-03-02T23:59:59Z",
        "bankroll_units": 105.8,
        "bankroll_dollars": 10580.0,
        "total_bets": 20,
        "total_wins": 12,
        "total_losses": 8,
        "win_rate": 0.6,
        "roi": 0.058,
        "units_won": 5.8,
        "avg_clv": 0.016
      }
    ]
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:35Z",
    "request_id": "req-uuid"
  }
}
```

**Consumers:** UI, CLI

---

### GET /api/v1/emulator/health

Health check for the bookie-emulator.

**Response:** `200 OK`

```json
{
  "data": {
    "status": "healthy",
    "service": "bookie-emulator",
    "version": "1.0.0",
    "uptime_seconds": 86400,
    "dependencies": {
      "database": "healthy",
      "redis": "healthy",
      "lines_service": "healthy",
      "statistics_service": "healthy"
    },
    "stats": {
      "open_bets": 3,
      "bets_today": 5,
      "bets_graded_today": 8
    }
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:35Z",
    "request_id": "req-uuid"
  }
}
```

**Consumers:** agent, infra-ops

---

## Events Published

None.

## Events Subscribed

| Event            | Channel                 | Purpose                                                            |
| ---------------- | ----------------------- | ------------------------------------------------------------------ |
| `game.completed` | `events:game.completed` | Triggers automated bet grading for open bets on the completed game |
