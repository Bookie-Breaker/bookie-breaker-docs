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

| Source | Data | Mechanism |
|---|---|---|
| simulation-engine | Outcome probability distributions (e.g., score distributions, margin distributions) | API call or internal pipeline |
| statistics-service | Contextual features: injury reports, rest days, travel, seasonal trends, matchup history | API call |
| lines-service | Line movement patterns, public betting percentages (if available), market-implied probabilities | API call |
| agent | Requests to generate predictions for specific games/bet types | API call |

## Outputs

| Destination | Data | Mechanism |
|---|---|---|
| agent | Calibrated probabilities for each bet type, confidence intervals, feature importance for each prediction | API response |
| CLI / UI / MCP server | Prediction summaries (may be accessed directly or through agent) | API response |

## Dependencies

- **simulation-engine** -- provides the base outcome distributions that get adjusted
- **statistics-service** -- provides contextual features for ML models
- **lines-service** -- provides line movement and market data as features

## Dependents

- **bookie-breaker-agent** -- consumes calibrated probabilities for edge detection and analysis
- **bookie-breaker-cli** -- may query predictions directly
- **bookie-breaker-ui** -- may display predictions directly
- **bookie-breaker-mcp-server** -- may expose predictions as MCP tools
