# Prediction Engine API

**Service:** prediction-engine
**Framework:** Python / FastAPI
**Base URL:** `/api/v1/predict`
**Port:** 8004

The prediction-engine applies ML-based adjustments to raw simulation distributions, accounting for contextual factors (injuries, rest, weather, line movement). It produces calibrated probabilities that the agent compares against market lines to identify edges.

---

## Endpoints

### POST /api/v1/predict/predictions

Generate calibrated predictions for a game across specified market types.

**Request Body:**

```json
{
  "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
  "simulation_run_id": "sr123456-789a-bcde-f012-3456789abcde",
  "market_types": ["SPREAD", "TOTAL", "MONEYLINE"]
}
```

| Field               | Type           | Required | Description                                                           |
| ------------------- | -------------- | -------- | --------------------------------------------------------------------- |
| `game_id`           | UUID           | Yes      | The game to predict.                                                  |
| `simulation_run_id` | UUID           | Yes      | The simulation run whose distributions to adjust.                     |
| `market_types`      | list\<string\> | No       | Market types to predict. Default: `["SPREAD", "TOTAL", "MONEYLINE"]`. |

**Headers:**

| Header              | Required | Description                     |
| ------------------- | -------- | ------------------------------- |
| `X-Idempotency-Key` | No       | UUID for idempotent submission. |

**Response:** `201 Created`

```json
{
  "data": {
    "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
    "simulation_run_id": "sr123456-789a-bcde-f012-3456789abcde",
    "predictions": [
      {
        "id": "pred1234-5678-9abc-def0-123456789abc",
        "market_type": "SPREAD",
        "selection": "LAL -3.5",
        "predicted_probability": 0.5312,
        "simulation_probability": 0.4812,
        "adjustment_magnitude": 0.05,
        "confidence_lower": 0.4912,
        "confidence_upper": 0.5712,
        "model_version_id": "mv123456-789a-bcde-f012-3456789abcde",
        "feature_importance": {
          "home_rest_days": 0.15,
          "away_injury_impact": 0.12,
          "line_movement_direction": 0.08,
          "home_away_split_diff": 0.07,
          "pace_differential": 0.06
        },
        "created_at": "2026-03-30T14:22:30Z"
      },
      {
        "id": "pred2345-6789-abcd-ef01-23456789abcd",
        "market_type": "TOTAL",
        "selection": "Over 220.5",
        "predicted_probability": 0.5823,
        "simulation_probability": 0.5534,
        "adjustment_magnitude": 0.0289,
        "confidence_lower": 0.5423,
        "confidence_upper": 0.6223,
        "model_version_id": "mv123456-789a-bcde-f012-3456789abcde",
        "feature_importance": {
          "pace_differential": 0.18,
          "combined_offensive_rating": 0.14,
          "rest_days_both": 0.09,
          "weather_impact": 0.0,
          "line_movement_direction": 0.07
        },
        "created_at": "2026-03-30T14:22:30Z"
      },
      {
        "id": "pred3456-789a-bcde-f012-3456789abcde",
        "market_type": "MONEYLINE",
        "selection": "LAL ML",
        "predicted_probability": 0.5645,
        "simulation_probability": 0.5412,
        "adjustment_magnitude": 0.0233,
        "confidence_lower": 0.5245,
        "confidence_upper": 0.6045,
        "model_version_id": "mv123456-789a-bcde-f012-3456789abcde",
        "feature_importance": {
          "home_court_advantage": 0.2,
          "net_rating_diff": 0.16,
          "away_injury_impact": 0.1,
          "head_to_head_recent": 0.05
        },
        "created_at": "2026-03-30T14:22:30Z"
      }
    ],
    "features_used": {
      "home_rest_days": 2,
      "away_rest_days": 1,
      "home_offensive_rating": 115.3,
      "away_offensive_rating": 118.1,
      "home_defensive_rating": 111.2,
      "away_defensive_rating": 108.5,
      "away_injury_impact": 0.0,
      "home_injury_impact": -2.5,
      "pace_differential": 1.7,
      "line_movement_direction": -1.0,
      "travel_distance_miles": 2451
    },
    "feature_source_versions": {
      "stats": "2026-03-30T12:00:00Z",
      "lines": "2026-03-30T14:18:00Z"
    }
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:30Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if game_id or simulation_run_id does not exist. `502 Bad Gateway` if statistics-service or lines-service is unavailable for feature retrieval.

**Consumers:** agent

---

### GET /api/v1/predict/predictions/{prediction_id}

Get a specific prediction with full feature details.

**Path Parameters:**

| Parameter       | Type | Description               |
| --------------- | ---- | ------------------------- |
| `prediction_id` | UUID | The prediction identifier |

**Response:** `200 OK`

```json
{
  "data": {
    "id": "pred1234-5678-9abc-def0-123456789abc",
    "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
    "simulation_run_id": "sr123456-789a-bcde-f012-3456789abcde",
    "model_version_id": "mv123456-789a-bcde-f012-3456789abcde",
    "market_type": "SPREAD",
    "selection": "LAL -3.5",
    "predicted_probability": 0.5312,
    "simulation_probability": 0.4812,
    "adjustment_magnitude": 0.05,
    "confidence_lower": 0.4912,
    "confidence_upper": 0.5712,
    "feature_importance": {
      "home_rest_days": 0.15,
      "away_injury_impact": 0.12,
      "line_movement_direction": 0.08
    },
    "feature_vector": {
      "id": "fv123456-789a-bcde-f012-3456789abcde",
      "features": {
        "home_rest_days": 2,
        "away_rest_days": 1,
        "home_offensive_rating": 115.3,
        "away_injury_impact": 0.0,
        "home_injury_impact": -2.5,
        "pace_differential": 1.7,
        "line_movement_direction": -1.0
      },
      "feature_source_versions": {
        "stats": "2026-03-30T12:00:00Z",
        "lines": "2026-03-30T14:18:00Z"
      }
    },
    "created_at": "2026-03-30T14:22:30Z"
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:35Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if prediction_id does not exist.

**Consumers:** agent, CLI, UI, MCP

---

### GET /api/v1/predict/games/{game_id}/latest

Get the most recent predictions for a game across all market types.

**Path Parameters:**

| Parameter | Type | Description         |
| --------- | ---- | ------------------- |
| `game_id` | UUID | The game identifier |

**Query Parameters:**

| Parameter       | Type   | Required | Default | Description                                          |
| --------------- | ------ | -------- | ------- | ---------------------------------------------------- |
| `market_type`   | string | No       | all     | Filter by market type. Comma-separated for multiple. |
| `model_version` | UUID   | No       | active  | Filter by specific model version.                    |

**Response:** `200 OK`

```json
{
  "data": {
    "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
    "predictions": [
      {
        "id": "pred1234-5678-9abc-def0-123456789abc",
        "market_type": "SPREAD",
        "selection": "LAL -3.5",
        "predicted_probability": 0.5312,
        "simulation_probability": 0.4812,
        "adjustment_magnitude": 0.05,
        "confidence_lower": 0.4912,
        "confidence_upper": 0.5712,
        "model_version_id": "mv123456-789a-bcde-f012-3456789abcde",
        "created_at": "2026-03-30T14:22:30Z"
      },
      {
        "id": "pred2345-6789-abcd-ef01-23456789abcd",
        "market_type": "TOTAL",
        "selection": "Over 220.5",
        "predicted_probability": 0.5823,
        "simulation_probability": 0.5534,
        "adjustment_magnitude": 0.0289,
        "confidence_lower": 0.5423,
        "confidence_upper": 0.6223,
        "model_version_id": "mv123456-789a-bcde-f012-3456789abcde",
        "created_at": "2026-03-30T14:22:30Z"
      }
    ]
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:35Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if no predictions exist for this game.

**Consumers:** agent, CLI, UI, MCP

---

### GET /api/v1/predict/games/{game_id}/edges

Get identified edges for a game. Edges are computed by comparing predictions against current market lines. This endpoint is a convenience that combines prediction data with line data.

**Path Parameters:**

| Parameter | Type | Description         |
| --------- | ---- | ------------------- |
| `game_id` | UUID | The game identifier |

**Query Parameters:**

| Parameter     | Type   | Required | Default | Description                         |
| ------------- | ------ | -------- | ------- | ----------------------------------- |
| `min_edge`    | float  | No       | 0.0     | Minimum edge percentage to include. |
| `market_type` | string | No       | all     | Filter by market type.              |

**Response:** `200 OK`

```json
{
  "data": {
    "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
    "edges": [
      {
        "prediction_id": "pred1234-5678-9abc-def0-123456789abc",
        "market_type": "SPREAD",
        "selection": "LAL -3.5",
        "predicted_probability": 0.5312,
        "implied_probability": 0.5238,
        "edge_percentage": 0.74,
        "best_odds_american": -110,
        "sportsbook_key": "draftkings"
      },
      {
        "prediction_id": "pred2345-6789-abcd-ef01-23456789abcd",
        "market_type": "TOTAL",
        "selection": "Over 220.5",
        "predicted_probability": 0.5823,
        "implied_probability": 0.5192,
        "edge_percentage": 6.31,
        "best_odds_american": -108,
        "sportsbook_key": "fanduel"
      }
    ]
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:35Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if no predictions exist for this game.

**Consumers:** agent, CLI, UI, MCP

---

### GET /api/v1/predict/models

List all model versions.

**Query Parameters:**

| Parameter     | Type    | Required | Default | Description                                       |
| ------------- | ------- | -------- | ------- | ------------------------------------------------- |
| `sport`       | string  | No       | all     | Filter by sport (e.g., `BASKETBALL`, `FOOTBALL`). |
| `market_type` | string  | No       | all     | Filter by market type.                            |
| `is_active`   | boolean | No       | (none)  | Filter by active status.                          |

**Response:** `200 OK`

```json
{
  "data": [
    {
      "id": "mv123456-789a-bcde-f012-3456789abcde",
      "sport": "BASKETBALL",
      "market_type": "SPREAD",
      "version_tag": "v2.1",
      "algorithm": "xgboost",
      "training_date": "2026-03-15T08:00:00Z",
      "training_samples": 45000,
      "evaluation_metrics": {
        "brier_score": 0.21,
        "log_loss": 0.68,
        "calibration_error": 0.018,
        "roi_backtest": 0.032
      },
      "is_active": true,
      "notes": "Improved injury impact features"
    }
  ],
  "meta": {
    "timestamp": "2026-03-30T14:22:35Z",
    "request_id": "req-uuid"
  }
}
```

**Consumers:** agent

---

### GET /api/v1/predict/models/{model_id}

Get detailed information about a specific model version.

**Path Parameters:**

| Parameter  | Type | Description                  |
| ---------- | ---- | ---------------------------- |
| `model_id` | UUID | The model version identifier |

**Response:** `200 OK`

```json
{
  "data": {
    "id": "mv123456-789a-bcde-f012-3456789abcde",
    "sport": "BASKETBALL",
    "market_type": "SPREAD",
    "version_tag": "v2.1",
    "algorithm": "xgboost",
    "training_date": "2026-03-15T08:00:00Z",
    "training_samples": 45000,
    "evaluation_metrics": {
      "brier_score": 0.21,
      "log_loss": 0.68,
      "calibration_error": 0.018,
      "roi_backtest": 0.032
    },
    "feature_names": [
      "home_rest_days",
      "away_rest_days",
      "home_offensive_rating",
      "away_offensive_rating",
      "home_defensive_rating",
      "away_defensive_rating",
      "home_injury_impact",
      "away_injury_impact",
      "pace_differential",
      "line_movement_direction",
      "travel_distance_miles",
      "home_away_split_diff",
      "head_to_head_recent"
    ],
    "is_active": true,
    "notes": "Improved injury impact features"
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:35Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if model_id does not exist.

**Consumers:** agent, CLI, UI

---

### POST /api/v1/predict/models/retrain

Trigger model retraining for a specific sport and market type.

**Request Body:**

```json
{
  "sport": "BASKETBALL",
  "market_type": "SPREAD",
  "training_config": {
    "min_samples": 1000,
    "test_split": 0.2,
    "date_from": "2024-01-01",
    "date_to": "2026-03-29"
  }
}
```

| Field                         | Type   | Required | Description                                       |
| ----------------------------- | ------ | -------- | ------------------------------------------------- |
| `sport`                       | string | Yes      | Sport enum value.                                 |
| `market_type`                 | string | Yes      | Market type enum value.                           |
| `training_config`             | object | No       | Training configuration overrides.                 |
| `training_config.min_samples` | int    | No       | Minimum training samples required. Default: 1000. |
| `training_config.test_split`  | float  | No       | Test set fraction. Default: 0.2.                  |
| `training_config.date_from`   | string | No       | Training data start date.                         |
| `training_config.date_to`     | string | No       | Training data end date.                           |

**Response:** `202 Accepted`

```json
{
  "data": {
    "retrain_id": "rt123456-789a-bcde-f012-3456789abcde",
    "sport": "BASKETBALL",
    "market_type": "SPREAD",
    "status": "started",
    "started_at": "2026-03-30T14:22:35Z",
    "estimated_duration_minutes": 15
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:35Z",
    "request_id": "req-uuid"
  }
}
```

**Consumers:** agent

---

### GET /api/v1/predict/health

Health check for the prediction-engine.

**Response:** `200 OK`

```json
{
  "data": {
    "status": "healthy",
    "service": "prediction-engine",
    "version": "1.0.0",
    "uptime_seconds": 86400,
    "dependencies": {
      "statistics_service": "healthy",
      "lines_service": "healthy",
      "redis": "healthy"
    },
    "active_models": {
      "BASKETBALL_SPREAD": "mv123456-789a-bcde-f012-3456789abcde",
      "BASKETBALL_TOTAL": "mv234567-89ab-cdef-0123-456789abcdef",
      "BASKETBALL_MONEYLINE": "mv345678-9abc-def0-1234-56789abcdef0"
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

| Event                  | Channel                       | Trigger                                    |
| ---------------------- | ----------------------------- | ------------------------------------------ |
| `prediction.completed` | `events:prediction.completed` | Predictions generated for a batch of games |

## Events Subscribed

None.
