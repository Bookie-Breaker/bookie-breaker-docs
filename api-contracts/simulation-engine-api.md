# Simulation Engine API

**Service:** simulation-engine
**Framework:** Python / FastAPI
**Base URL:** `/api/v1/sim`
**Port:** 8003

The simulation-engine runs Monte Carlo simulations to generate probability distributions for game outcomes. It uses
sport-specific plugins (football, basketball, baseball) to model game mechanics and produces full outcome distributions
that feed into the prediction-engine for contextual adjustment.

---

## Endpoints

### POST /api/v1/sim/simulations

Run a Monte Carlo simulation for a specific game.

**Request Body:**

```json
{
  "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
  "config": {
    "iterations": 10000,
    "convergence_threshold": 0.005,
    "random_seed": null,
    "plugin_config": {}
  },
  "force_refresh": false
}
```

| Field                          | Type    | Required | Description                                                 |
| ------------------------------ | ------- | -------- | ----------------------------------------------------------- |
| `game_id`                      | UUID    | Yes      | The game to simulate.                                       |
| `config`                       | object  | No       | Simulation config overrides.                                |
| `config.iterations`            | int     | No       | Number of iterations (default: 10000, max: 50000).          |
| `config.convergence_threshold` | float   | No       | Stop early if distributions converge within this tolerance. |
| `config.random_seed`           | int     | No       | Seed for reproducibility. Null for random.                  |
| `config.plugin_config`         | object  | No       | Sport-specific parameters.                                  |
| `force_refresh`                | boolean | No       | If true, ignore cached results and re-run. Default: false.  |

**Headers:**

| Header              | Required | Description                     |
| ------------------- | -------- | ------------------------------- |
| `X-Idempotency-Key` | No       | UUID for idempotent submission. |

**Response:** `201 Created`

```json
{
  "data": {
    "simulation_run_id": "sr123456-789a-bcde-f012-3456789abcde",
    "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
    "status": "completed",
    "cached": false,
    "config": {
      "sport": "BASKETBALL",
      "iterations": 10000,
      "convergence_threshold": 0.005,
      "random_seed": null
    },
    "started_at": "2026-03-30T14:22:05Z",
    "completed_at": "2026-03-30T14:22:28Z",
    "duration_ms": 23000,
    "iterations_completed": 10000,
    "converged": true,
    "parameters_hash": "a1b2c3d4e5f6",
    "result": {
      "id": "res12345-6789-abcd-ef01-234567890abc",
      "home_win_probability": 0.5412,
      "away_win_probability": 0.4588,
      "draw_probability": 0.0,
      "mean_home_score": 112.4,
      "mean_away_score": 109.8,
      "mean_total": 222.2,
      "mean_margin": 2.6,
      "spread_cover_probabilities": {
        "-3.5": 0.4812,
        "-2.5": 0.5198,
        "-1.5": 0.5342,
        "+3.5": 0.5188,
        "+2.5": 0.4802
      },
      "total_over_probabilities": {
        "218.5": 0.6123,
        "220.5": 0.5534,
        "222.5": 0.4812,
        "224.5": 0.4102
      },
      "percentiles": {
        "margin": { "10": -12, "25": -5, "50": 3, "75": 10, "90": 17 },
        "total": { "10": 200, "25": 211, "50": 222, "75": 233, "90": 244 }
      }
    }
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:28Z",
    "request_id": "req-uuid"
  }
}
```

If cached results are returned (identical `parameters_hash` found and `force_refresh` is false), the response will have
`"cached": true` and the original timestamps/duration.

**Error:** `404 Not Found` if game_id does not exist in statistics-service. `422 Unprocessable Entity` if game is
already completed or cancelled. `502 Bad Gateway` if statistics-service is unavailable.

**Consumers:** agent

---

### GET /api/v1/sim/simulations/{simulation_id}

Get results of a specific simulation run.

**Path Parameters:**

| Parameter       | Type | Description                   |
| --------------- | ---- | ----------------------------- |
| `simulation_id` | UUID | The simulation run identifier |

**Response:** `200 OK`

```json
{
  "data": {
    "simulation_run_id": "sr123456-789a-bcde-f012-3456789abcde",
    "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
    "status": "completed",
    "config": {
      "sport": "BASKETBALL",
      "iterations": 10000,
      "convergence_threshold": 0.005,
      "random_seed": null
    },
    "started_at": "2026-03-30T14:22:05Z",
    "completed_at": "2026-03-30T14:22:28Z",
    "duration_ms": 23000,
    "iterations_completed": 10000,
    "converged": true,
    "parameters_hash": "a1b2c3d4e5f6",
    "batch_id": null,
    "result": {
      "id": "res12345-6789-abcd-ef01-234567890abc",
      "home_win_probability": 0.5412,
      "away_win_probability": 0.4588,
      "draw_probability": 0.0,
      "mean_home_score": 112.4,
      "mean_away_score": 109.8,
      "mean_total": 222.2,
      "mean_margin": 2.6,
      "spread_cover_probabilities": {
        "-3.5": 0.4812,
        "-2.5": 0.5198,
        "+3.5": 0.5188,
        "+2.5": 0.4802
      },
      "total_over_probabilities": {
        "220.5": 0.5534,
        "222.5": 0.4812
      },
      "percentiles": {
        "margin": { "10": -12, "25": -5, "50": 3, "75": 10, "90": 17 },
        "total": { "10": 200, "25": 211, "50": 222, "75": 233, "90": 244 }
      }
    }
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:30Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if simulation_id does not exist.

**Consumers:** agent, prediction-engine, CLI, UI, MCP

---

### POST /api/v1/sim/simulations/batch

Batch simulate multiple games in a single request. Games are processed in parallel.

**Request Body:**

```json
{
  "games": [
    {
      "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
      "config": {
        "iterations": 10000
      }
    },
    {
      "game_id": "g2345678-9abc-def0-1234-56789abcdef0",
      "config": {
        "iterations": 10000
      }
    }
  ],
  "default_config": {
    "iterations": 10000,
    "convergence_threshold": 0.005
  },
  "force_refresh": false
}
```

| Field             | Type           | Required | Description                                            |
| ----------------- | -------------- | -------- | ------------------------------------------------------ |
| `games`           | list\<object\> | Yes      | List of games to simulate.                             |
| `games[].game_id` | UUID           | Yes      | Game identifier.                                       |
| `games[].config`  | object         | No       | Per-game config overrides.                             |
| `default_config`  | object         | No       | Default config applied to all games unless overridden. |
| `force_refresh`   | boolean        | No       | Ignore cached results. Default: false.                 |

**Response:** `201 Created`

```json
{
  "data": {
    "batch_id": "batch123-4567-89ab-cdef-0123456789ab",
    "status": "completed",
    "total_games": 2,
    "completed_games": 2,
    "failed_games": 0,
    "started_at": "2026-03-30T14:22:05Z",
    "completed_at": "2026-03-30T14:22:48Z",
    "total_duration_ms": 43000,
    "results": [
      {
        "simulation_run_id": "sr123456-789a-bcde-f012-3456789abcde",
        "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
        "status": "completed",
        "cached": false,
        "result": {
          "home_win_probability": 0.5412,
          "away_win_probability": 0.4588,
          "mean_total": 222.2,
          "mean_margin": 2.6
        }
      },
      {
        "simulation_run_id": "sr234567-89ab-cdef-0123-456789abcdef",
        "game_id": "g2345678-9abc-def0-1234-56789abcdef0",
        "status": "completed",
        "cached": true,
        "result": {
          "home_win_probability": 0.6234,
          "away_win_probability": 0.3766,
          "mean_total": 198.5,
          "mean_margin": 7.1
        }
      }
    ]
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:48Z",
    "request_id": "req-uuid"
  }
}
```

**Consumers:** agent

---

### GET /api/v1/sim/games/{game_id}/latest

Get the most recent simulation results for a game.

**Path Parameters:**

| Parameter | Type | Description         |
| --------- | ---- | ------------------- |
| `game_id` | UUID | The game identifier |

**Query Parameters:**

| Parameter       | Type    | Required | Default | Description                                                    |
| --------------- | ------- | -------- | ------- | -------------------------------------------------------------- |
| `config_id`     | UUID    | No       | (none)  | Filter by specific simulation config.                          |
| `force_refresh` | boolean | No       | `false` | If true, trigger a new simulation instead of returning cached. |

**Response:** `200 OK`

Same structure as `GET /api/v1/sim/simulations/{simulation_id}`.

**Error:** `404 Not Found` if no simulations exist for this game.

**Consumers:** prediction-engine, agent, CLI, UI, MCP

---

### GET /api/v1/sim/simulations/{simulation_id}/distributions

Get detailed raw distribution data for a simulation, suitable for visualization.

**Path Parameters:**

| Parameter       | Type | Description                   |
| --------------- | ---- | ----------------------------- |
| `simulation_id` | UUID | The simulation run identifier |

**Query Parameters:**

| Parameter           | Type   | Required | Default | Description                                                                          |
| ------------------- | ------ | -------- | ------- | ------------------------------------------------------------------------------------ |
| `distribution_type` | string | No       | `all`   | Which distributions to return: `margin`, `total`, `home_score`, `away_score`, `all`. |

**Response:** `200 OK`

```json
{
  "data": {
    "simulation_run_id": "sr123456-789a-bcde-f012-3456789abcde",
    "game_id": "g1234567-89ab-cdef-0123-456789abcdef",
    "iterations_completed": 10000,
    "distributions": {
      "home_score": {
        "type": "discrete",
        "values": {
          "95": 0.012,
          "96": 0.014,
          "97": 0.018,
          "98": 0.022,
          "99": 0.028,
          "100": 0.035,
          "101": 0.041,
          "102": 0.048,
          "103": 0.054,
          "104": 0.058,
          "105": 0.062,
          "106": 0.063,
          "107": 0.061,
          "108": 0.058,
          "109": 0.052,
          "110": 0.047,
          "111": 0.041,
          "112": 0.035,
          "113": 0.03,
          "114": 0.025,
          "115": 0.02,
          "116": 0.016,
          "117": 0.012,
          "118": 0.009
        },
        "mean": 112.4,
        "std_dev": 8.2,
        "min": 85,
        "max": 145
      },
      "away_score": {
        "type": "discrete",
        "values": {},
        "mean": 109.8,
        "std_dev": 8.5,
        "min": 82,
        "max": 142
      },
      "margin": {
        "type": "discrete",
        "values": {},
        "mean": 2.6,
        "std_dev": 12.1,
        "min": -35,
        "max": 38
      },
      "total": {
        "type": "discrete",
        "values": {},
        "mean": 222.2,
        "std_dev": 11.8,
        "min": 178,
        "max": 270
      }
    }
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:30Z",
    "request_id": "req-uuid"
  }
}
```

**Error:** `404 Not Found` if simulation_id does not exist.

**Consumers:** UI (charts), CLI

---

### GET /api/v1/sim/health

Health check with current load information.

**Response:** `200 OK`

```json
{
  "data": {
    "status": "healthy",
    "service": "simulation-engine",
    "version": "1.0.0",
    "uptime_seconds": 86400,
    "dependencies": {
      "statistics_service": "healthy",
      "redis": "healthy"
    },
    "load": {
      "active_simulations": 2,
      "queued_simulations": 0,
      "max_concurrent": 30,
      "simulations_today": 145,
      "avg_duration_ms": 22500
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

| Event                  | Channel                       | Trigger                       |
| ---------------------- | ----------------------------- | ----------------------------- |
| `simulation.completed` | `events:simulation.completed` | Batch of simulations finishes |

## Events Subscribed

None.
