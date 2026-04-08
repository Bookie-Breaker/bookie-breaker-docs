# bookie-breaker-prediction-engine

## Purpose

ML adjustment layer that takes raw simulation output distributions and applies machine learning models to adjust for contextual factors the simulation does not capture (injuries, weather, rest days, public betting percentages, line movement patterns). Produces final calibrated probabilities that are compared against market lines to identify edges.

## Responsibilities

- Receives outcome distributions from the simulation-engine and applies ML-based adjustments.
- Owns sport-specific feature engineering: transforming raw contextual data into model features (e.g., rest days between games, travel distance, injury impact scores, weather effects on totals).
- Trains and maintains gradient boosting models (baseline) with potential for sport-specific model variants.
- Produces calibrated probability outputs -- not just predictions, but well-calibrated probabilities that accurately reflect true outcome likelihood.
- Owns model evaluation: tracking calibration, Brier scores, log loss, and other probabilistic scoring metrics over time.
- Manages model versioning and A/B testing of model variants.
- Applies adjustments for all supported bet types (spreads, totals, moneylines, props) across all 6 leagues.

## Non-Responsibilities

- Does NOT run simulations. It receives simulation output from the simulation-engine.
- Does NOT collect or store raw statistics. It requests features from the statistics-service and lines-service.
- Does NOT detect edges. It produces calibrated probabilities; the agent compares those against market lines.
- Does NOT generate natural language analysis. The agent handles interpretation.
- Does NOT place or track bets. The bookie-emulator handles that.
- Does NOT own the raw data pipelines. statistics-service and lines-service own ingestion.

## Inputs

| Source             | Data                                                                                            | Mechanism                     |
| ------------------ | ----------------------------------------------------------------------------------------------- | ----------------------------- |
| simulation-engine  | Outcome probability distributions (e.g., score distributions, margin distributions)             | API call or internal pipeline |
| statistics-service | Contextual features: injury reports, rest days, travel, seasonal trends, matchup history        | API call                      |
| lines-service      | Line movement patterns, public betting percentages (if available), market-implied probabilities | API call                      |
| agent              | Requests to generate predictions for specific games/bet types                                   | API call                      |

## Outputs

| Destination           | Data                                                                                                     | Mechanism    |
| --------------------- | -------------------------------------------------------------------------------------------------------- | ------------ |
| agent                 | Calibrated probabilities for each bet type, confidence intervals, feature importance for each prediction | API response |
| CLI / UI / MCP server | Prediction summaries (may be accessed directly or through agent)                                         | API response |

## Dependencies

- **simulation-engine** -- provides the base outcome distributions that get adjusted
- **statistics-service** -- provides contextual features for ML models
- **lines-service** -- provides line movement and market data as features

## Dependents

- **bookie-breaker-agent** -- consumes calibrated probabilities for edge detection and analysis
- **bookie-breaker-cli** -- may query predictions directly
- **bookie-breaker-ui** -- may display predictions directly
- **bookie-breaker-mcp-server** -- may expose predictions as MCP tools

---

## Requirements

### Functional Requirements

- **FR-001:** Accept outcome distributions from the simulation-engine and apply ML-based adjustments for contextual factors not captured by the simulation (injuries, rest days, travel distance, weather, public betting percentages, line movement patterns).
- **FR-002:** Own sport-specific feature engineering pipelines that transform raw contextual data into model features (e.g., rest_days_diff, injury_impact_score, travel_distance_miles, home_away_split_differential, line_movement_direction, weather_wind_speed).
- **FR-003:** Train and maintain gradient boosting models (XGBoost or LightGBM) as the baseline algorithm, with support for sport-specific and market-specific model variants (e.g., separate models for NFL spreads, NBA totals, MLB moneylines).
- **FR-004:** Produce calibrated probability outputs where predicted probabilities accurately reflect true outcome likelihood (calibration error < 0.03 on holdout data).
- **FR-005:** Return confidence intervals (lower and upper bounds) for every predicted probability to quantify prediction uncertainty.
- **FR-006:** Return feature importance scores for each prediction, enabling explainability (e.g., "injury adjustment contributed +3% to the prediction").
- **FR-007:** Apply ML adjustments for all supported bet types (spreads, totals, moneylines, player props, team props) across all 6 leagues.
- **FR-008:** Track and report model evaluation metrics over time: Brier score, log loss, calibration error, and ROI achieved by the model's predictions.
- **FR-009:** Support model versioning: maintain a registry of ModelVersions with training metadata, evaluation metrics, and active/inactive status. Only one model per sport/market_type combination may be active at a time.
- **FR-010:** Support A/B testing of model variants by running multiple model versions against the same inputs and comparing outputs.
- **FR-011:** Store every Prediction with its associated FeatureVector for post-hoc analysis, debugging, and model retraining.
- **FR-012:** Retrain models on a configurable schedule (at minimum monthly) using accumulated historical data from simulation results, actual outcomes, and feature vectors.
- **FR-013:** Publish a `prediction.completed` event to `events:prediction.completed` when predictions are generated for a batch of games.

### Non-Functional Requirements

- **Latency:** ML inference (feature engineering + model prediction) for a single game across all market types: < 5 seconds. Full batch prediction for a day's games (5-30 games): < 2 minutes. API response for stored predictions: < 200ms.
- **Throughput:** Process up to 30 game predictions per batch run. Handle up to 50 API requests/second for stored prediction lookups from agent and interfaces.
- **Availability:** 99% uptime. Graceful degradation: if statistics-service or lines-service is temporarily unavailable for feature retrieval, use most recent cached feature values and flag predictions as "stale features" in the response.
- **Storage:** Each Prediction + FeatureVector is ~2-5 KB. Estimated 200,000-500,000 predictions/year (games x market types). Total: ~1-2 GB/year. Model artifacts: ~50-200 MB per trained model.

### Data Ownership

This service is the source of truth for:

- **Prediction** -- calibrated probabilities for each game/market, including raw vs. adjusted probabilities, confidence intervals, and feature importance.
- **ModelVersion** -- metadata about trained model versions (algorithm, training date, evaluation metrics, active status).
- **FeatureVector** -- input features used for each prediction, stored for explainability and retraining.

### APIs Exposed

| Method + Path                                   | Description                                | Key Query Parameters                             | Consumers           |
| ----------------------------------------------- | ------------------------------------------ | ------------------------------------------------ | ------------------- |
| `POST /api/v1/predictions`                      | Generate predictions for one or more games | Body: game_ids, simulation_run_ids, market_types | agent               |
| `GET /api/v1/predictions/{prediction_id}`       | Get a specific prediction with features    | --                                               | agent, CLI, UI, MCP |
| `GET /api/v1/predictions/game/{game_id}`        | Get all predictions for a game             | `market_type`, `model_version`                   | agent, CLI, UI, MCP |
| `GET /api/v1/predictions/recent`                | Get recent predictions                     | `league`, `date`, `market_type`, `limit`         | CLI, UI, MCP        |
| `GET /api/v1/models`                            | List model versions                        | `sport`, `market_type`, `is_active`              | agent               |
| `GET /api/v1/models/{model_version_id}/metrics` | Get evaluation metrics for a model         | --                                               | agent, CLI, UI      |
| `POST /api/v1/models/retrain`                   | Trigger model retraining                   | Body: sport, market_type, training_config        | agent               |

### APIs Consumed

| Service            | Endpoint                                            | Purpose                                                      |
| ------------------ | --------------------------------------------------- | ------------------------------------------------------------ |
| simulation-engine  | `GET /api/v1/simulations/game/{game_id}`            | Get base outcome distributions to adjust                     |
| statistics-service | `GET /api/v1/injuries`                              | Get injury reports for injury impact features                |
| statistics-service | `GET /api/v1/games`                                 | Get schedule data for rest days, travel distance computation |
| statistics-service | `GET /api/v1/teams/{team_id}/stats`                 | Get team home/away splits, seasonal trends                   |
| statistics-service | `GET /api/v1/players/{player_id}/stats`             | Get player status and impact data                            |
| statistics-service | `GET /api/v1/venues/{venue_id}`                     | Get venue details for weather relevance, travel distance     |
| statistics-service | `GET /api/v1/matchup/{home_team_id}/{away_team_id}` | Get matchup history for head-to-head features                |
| lines-service      | `GET /api/v1/lines/{game_id}/movement`              | Get line movement patterns for sharp money features          |
| lines-service      | `GET /api/v1/lines/{game_id}`                       | Get current market-implied probabilities                     |

### Events Published

| Event                  | Channel                       | Description                                                                                                                          |
| ---------------------- | ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `prediction.completed` | `events:prediction.completed` | Published when predictions are generated for a batch. Payload includes batch_id, game_ids, league, edges_found count, and bet_types. |

### Events Subscribed

None. The prediction-engine runs predictions on demand via synchronous API calls from the agent.

### Storage Requirements

- **Database:** PostgreSQL.
- **Key tables:**
  - `model_versions` -- model registry (~50-100 rows over time, mostly historical).
  - `predictions` -- calibrated prediction outputs (~200K-500K rows/year).
  - `feature_vectors` -- input features per prediction (~200K-500K rows/year, stored as JSONB).
- **Estimated row counts:** Year 1: ~200K-500K predictions + feature vectors. Year 3: ~1M-1.5M total.
- **Growth:** ~1,000-3,000 new predictions/day during active seasons.
- **Indexing priorities:**
  1. `predictions(game_id, market_type, created_at DESC)` -- most recent prediction for a game/market.
  2. `predictions(model_version_id)` -- model performance analysis.
  3. `predictions(created_at)` -- time-range queries for retraining data.
- **Model artifact storage:** Model files (XGBoost/LightGBM serialized models) stored on mounted volume, ~50-200 MB per model. Keep last 5 versions per sport/market_type.
- **Redis:** Used for feature cache (15m TTL) to avoid redundant statistics-service/lines-service calls within a prediction batch.
