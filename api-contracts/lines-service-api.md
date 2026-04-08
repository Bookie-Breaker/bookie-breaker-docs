# Lines Service API

**Service:** lines-service
**Framework:** Go / Echo
**Base URL:** `/api/v1/lines`
**Port:** 8001

The lines-service ingests, normalizes, stores, and serves betting lines from external odds APIs. It is the system's
source of truth for current and historical lines across all sportsbooks, bet types, and leagues.

---

## Endpoints

### GET /api/v1/lines/current

List current betting lines with optional filters. Returns the most recent line snapshot per game/sportsbook/market
combination.

**Query Parameters:**

| Parameter     | Type   | Required | Default | Description                                                                                      |
| ------------- | ------ | -------- | ------- | ------------------------------------------------------------------------------------------------ |
| `league`      | string | No       | all     | Filter by league (e.g., `NBA`, `NFL`). Comma-separated for multiple.                             |
| `game_id`     | string | No       | (none)  | Filter by specific game UUID. Comma-separated for multiple.                                      |
| `sportsbook`  | string | No       | all     | Filter by sportsbook key (e.g., `draftkings`). Comma-separated for multiple.                     |
| `market_type` | string | No       | all     | Filter by market type enum (e.g., `SPREAD`, `TOTAL`, `MONEYLINE`). Comma-separated for multiple. |
| `date`        | string | No       | today   | Filter by game date (ISO 8601 date, e.g., `2026-03-30`).                                         |
| `limit`       | int    | No       | 50      | Max results per page (max 200).                                                                  |
| `cursor`      | string | No       | (none)  | Pagination cursor.                                                                               |

**Response:** `200 OK`

```json
{
  "data": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
      "sportsbook_id": "sb123456-789a-bcde-f012-3456789abcde",
      "sportsbook_key": "draftkings",
      "market_type": "SPREAD",
      "selection": "LAL -3.5",
      "side": "HOME",
      "line_value": -3.5,
      "odds_american": -110,
      "odds_decimal": 1.909,
      "implied_probability": 0.5238,
      "timestamp": "2026-03-30T14:22:00Z",
      "is_opening": false,
      "is_closing": false
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

**Consumers:** agent, prediction-engine, bookie-emulator, CLI, UI, MCP

---

### GET /api/v1/lines/snapshots/{line_id}

Get a specific line snapshot by its UUID.

**Path Parameters:**

| Parameter | Type | Description                  |
| --------- | ---- | ---------------------------- |
| `line_id` | UUID | The line snapshot identifier |

**Response:** `200 OK`

```json
{
  "data": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
    "sportsbook_id": "sb123456-789a-bcde-f012-3456789abcde",
    "sportsbook_key": "draftkings",
    "sportsbook_name": "DraftKings",
    "market_type": "SPREAD",
    "selection": "LAL -3.5",
    "side": "HOME",
    "line_value": -3.5,
    "odds_american": -110,
    "odds_decimal": 1.909,
    "implied_probability": 0.5238,
    "timestamp": "2026-03-30T14:22:00Z",
    "is_opening": false,
    "is_closing": false,
    "player_id": null,
    "stat_type": null
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:05Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if line_id does not exist.

**Consumers:** agent, bookie-emulator

---

### GET /api/v1/lines/game/{game_id}

Get all current lines for a specific game across all sportsbooks and market types.

**Path Parameters:**

| Parameter | Type | Description         |
| --------- | ---- | ------------------- |
| `game_id` | UUID | The game identifier |

**Query Parameters:**

| Parameter     | Type   | Required | Default | Description                                             |
| ------------- | ------ | -------- | ------- | ------------------------------------------------------- |
| `market_type` | string | No       | all     | Filter by market type. Comma-separated for multiple.    |
| `sportsbook`  | string | No       | all     | Filter by sportsbook key. Comma-separated for multiple. |
| `side`        | string | No       | all     | Filter by side (e.g., `HOME`, `AWAY`, `OVER`, `UNDER`). |
| `limit`       | int    | No       | 50      | Max results per page.                                   |
| `cursor`      | string | No       | (none)  | Pagination cursor.                                      |

**Response:** `200 OK`

```json
{
  "data": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
      "sportsbook_id": "sb123456-789a-bcde-f012-3456789abcde",
      "sportsbook_key": "draftkings",
      "market_type": "SPREAD",
      "selection": "LAL -3.5",
      "side": "HOME",
      "line_value": -3.5,
      "odds_american": -110,
      "odds_decimal": 1.909,
      "implied_probability": 0.5238,
      "timestamp": "2026-03-30T14:20:00Z",
      "is_opening": false,
      "is_closing": false
    },
    {
      "id": "b2c3d4e5-f678-9012-bcde-f12345678901",
      "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
      "sportsbook_id": "sb234567-89ab-cdef-0123-456789abcdef",
      "sportsbook_key": "fanduel",
      "market_type": "SPREAD",
      "selection": "LAL -3",
      "side": "HOME",
      "line_value": -3.0,
      "odds_american": -108,
      "odds_decimal": 1.926,
      "implied_probability": 0.5192,
      "timestamp": "2026-03-30T14:18:00Z",
      "is_opening": false,
      "is_closing": false
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

**Error:** `404 Not Found` if game_id does not exist.

**Consumers:** agent, prediction-engine, bookie-emulator, CLI, UI, MCP

---

### GET /api/v1/lines/game/{game_id}/movement

Get line movement history for a specific game. Returns the full sequence of line snapshots from opening to current,
aggregated into a LineMovement structure.

**Path Parameters:**

| Parameter | Type | Description         |
| --------- | ---- | ------------------- |
| `game_id` | UUID | The game identifier |

**Query Parameters:**

| Parameter     | Type   | Required | Default  | Description                             |
| ------------- | ------ | -------- | -------- | --------------------------------------- |
| `sportsbook`  | string | No       | all      | Filter by sportsbook key.               |
| `market_type` | string | No       | `SPREAD` | Market type for movement.               |
| `selection`   | string | No       | (none)   | Specific selection string to filter on. |

**Response:** `200 OK`

```json
{
  "data": [
    {
      "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
      "sportsbook_id": "sb123456-789a-bcde-f012-3456789abcde",
      "sportsbook_key": "draftkings",
      "market_type": "SPREAD",
      "selection": "LAL -3.5",
      "opening_line": -2.5,
      "opening_odds": -110,
      "current_line": -3.5,
      "current_odds": -110,
      "closing_line": null,
      "closing_odds": null,
      "total_movement": -1.0,
      "is_reverse_movement": false,
      "line_snapshots": [
        {
          "line_value": -2.5,
          "odds_american": -110,
          "timestamp": "2026-03-28T12:00:00Z",
          "is_opening": true
        },
        {
          "line_value": -3.0,
          "odds_american": -105,
          "timestamp": "2026-03-29T08:00:00Z",
          "is_opening": false
        },
        {
          "line_value": -3.5,
          "odds_american": -110,
          "timestamp": "2026-03-30T10:00:00Z",
          "is_opening": false
        }
      ]
    }
  ],
  "meta": {
    "timestamp": "2026-03-30T14:22:05Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if game_id does not exist.

**Consumers:** prediction-engine, agent, UI

---

### GET /api/v1/lines/game/{game_id}/best

Get the best available line per market type across all tracked sportsbooks.

**Path Parameters:**

| Parameter | Type | Description         |
| --------- | ---- | ------------------- |
| `game_id` | UUID | The game identifier |

**Query Parameters:**

| Parameter     | Type   | Required | Default | Description                                          |
| ------------- | ------ | -------- | ------- | ---------------------------------------------------- |
| `market_type` | string | No       | all     | Filter by market type. Comma-separated for multiple. |
| `selection`   | string | No       | (none)  | Specific selection to find best odds for.            |

**Response:** `200 OK`

```json
{
  "data": [
    {
      "market_type": "SPREAD",
      "selection": "LAL -3.5",
      "side": "HOME",
      "line_value": -3.5,
      "best_odds_american": -105,
      "best_odds_decimal": 1.952,
      "implied_probability": 0.5122,
      "sportsbook_id": "sb234567-89ab-cdef-0123-456789abcdef",
      "sportsbook_key": "pinnacle",
      "sportsbook_name": "Pinnacle",
      "timestamp": "2026-03-30T14:18:00Z",
      "line_id": "c3d4e5f6-7890-1234-cdef-012345678901"
    },
    {
      "market_type": "TOTAL",
      "selection": "Over 220.5",
      "side": "OVER",
      "line_value": 220.5,
      "best_odds_american": -108,
      "best_odds_decimal": 1.926,
      "implied_probability": 0.5192,
      "sportsbook_id": "sb345678-9abc-def0-1234-56789abcdef0",
      "sportsbook_key": "betmgm",
      "sportsbook_name": "BetMGM",
      "timestamp": "2026-03-30T14:15:00Z",
      "line_id": "d4e5f678-9012-3456-def0-123456789012"
    }
  ],
  "meta": {
    "timestamp": "2026-03-30T14:22:05Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if game_id does not exist.

**Consumers:** agent (edge detection)

---

### GET /api/v1/lines/sportsbooks

List all tracked sportsbooks.

**Query Parameters:**

| Parameter   | Type    | Required | Default | Description                           |
| ----------- | ------- | -------- | ------- | ------------------------------------- |
| `is_sharp`  | boolean | No       | (none)  | Filter by sharp/market-making status. |
| `is_active` | boolean | No       | `true`  | Filter by active status.              |

**Response:** `200 OK`

```json
{
  "data": [
    {
      "id": "sb123456-789a-bcde-f012-3456789abcde",
      "name": "DraftKings",
      "key": "draftkings",
      "is_sharp": false,
      "is_active": true
    },
    {
      "id": "sb234567-89ab-cdef-0123-456789abcdef",
      "name": "Pinnacle",
      "key": "pinnacle",
      "is_sharp": true,
      "is_active": true
    }
  ],
  "meta": {
    "timestamp": "2026-03-30T14:22:05Z",
    "request_id": "req-uuid"
  }
}
```

**Consumers:** agent, prediction-engine

---

### GET /api/v1/lines/game/{game_id}/closing

Get closing lines (final lines before game start) for a completed game.

**Path Parameters:**

| Parameter | Type | Description         |
| --------- | ---- | ------------------- |
| `game_id` | UUID | The game identifier |

**Query Parameters:**

| Parameter     | Type   | Required | Default | Description               |
| ------------- | ------ | -------- | ------- | ------------------------- |
| `sportsbook`  | string | No       | all     | Filter by sportsbook key. |
| `market_type` | string | No       | all     | Filter by market type.    |

**Response:** `200 OK`

```json
{
  "data": [
    {
      "id": "e5f67890-1234-5678-ef01-234567890123",
      "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
      "sportsbook_id": "sb123456-789a-bcde-f012-3456789abcde",
      "sportsbook_key": "draftkings",
      "market_type": "SPREAD",
      "selection": "LAL -4",
      "side": "HOME",
      "line_value": -4.0,
      "odds_american": -110,
      "odds_decimal": 1.909,
      "implied_probability": 0.5238,
      "timestamp": "2026-03-30T19:28:00Z",
      "is_opening": false,
      "is_closing": true
    }
  ],
  "meta": {
    "timestamp": "2026-03-30T22:15:00Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if game_id does not exist or no closing lines are available.

**Consumers:** bookie-emulator (CLV calculation)

---

### POST /api/v1/lines/ingestion/trigger

Manually trigger a lines ingestion cycle. Used for debugging and manual refresh.

**Request Body:**

```json
{
  "leagues": ["NBA", "NFL"],
  "sources": ["odds_api"]
}
```

| Field     | Type           | Required | Description                                            |
| --------- | -------------- | -------- | ------------------------------------------------------ |
| `leagues` | list\<string\> | No       | Leagues to ingest. Default: all active leagues.        |
| `sources` | list\<string\> | No       | Data sources to poll. Default: all configured sources. |

**Response:** `202 Accepted`

```json
{
  "data": {
    "ingestion_id": "ing12345-6789-abcd-ef01-234567890abc",
    "status": "started",
    "leagues": ["NBA", "NFL"],
    "sources": ["odds_api"],
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

### GET /api/v1/lines/health

Health check for the lines-service.

**Response:** `200 OK`

```json
{
  "data": {
    "status": "healthy",
    "service": "lines-service",
    "version": "1.0.0",
    "uptime_seconds": 86400,
    "dependencies": {
      "database": "healthy",
      "redis": "healthy",
      "odds_api": "healthy"
    },
    "last_ingestion": {
      "timestamp": "2026-03-30T14:20:00Z",
      "lines_ingested": 342,
      "leagues_polled": ["NBA", "MLB"]
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

| Event           | Channel                | Trigger                                     |
| --------------- | ---------------------- | ------------------------------------------- |
| `lines.updated` | `events:lines.updated` | New or changed lines detected and persisted |

## Events Subscribed

None.
