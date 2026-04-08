# Prediction Models

Algorithm design for BookieBreaker's ML adjustment layer. This document specifies the feature engineering, model
architecture, training pipeline, calibration methods, and model management for the prediction-engine.

---

## Table of Contents

1. [ML Adjustment Layer Architecture](#1-ml-adjustment-layer-architecture)
2. [Feature Engineering per Sport](#2-feature-engineering-per-sport)
3. [Model Architecture](#3-model-architecture)
4. [Training Pipeline](#4-training-pipeline)
5. [Calibration](#5-calibration)
6. [Model Versioning and A/B Testing](#6-model-versioning-and-ab-testing)

---

## 1. ML Adjustment Layer Architecture

### Why Adjustment Is Needed

The simulation-engine produces probability distributions based on season-level team and player statistics. These
distributions are mechanistically sound but miss factors that are difficult to encode in a physics-based simulation:

- **Recency:** A team that has won 8 of its last 10 plays differently than its season averages suggest
- **Injuries:** A starting quarterback being ruled out shifts spread expectations by 3-14 points
- **Weather:** Wind and temperature affect passing, kicking, and ball flight in ways that require game-specific
  adjustment
- **Rest and travel:** Back-to-back games, short weeks, and cross-country travel cause measurable performance drops
- **Market signals:** Sharp line movement indicates information the simulation does not have
- **Motivation and situational factors:** Rivalry games, elimination scenarios, and coaching changes alter team behavior

The ML layer takes simulation outputs as a strong prior and adjusts them using these contextual features.

### Input/Output Specification

**Input:**

```python
@dataclass
class PredictionInput:
    # From simulation-engine
    simulation_run_id: str
    home_win_prob: float
    away_win_prob: float
    margin_mean: float
    margin_std: float
    total_mean: float
    total_std: float
    spread_covers: dict[float, float]  # {line: P(home covers)}
    total_overs: dict[float, float]    # {line: P(over)}

    # Contextual features (from statistics-service, lines-service)
    features: dict[str, float | int | bool | None]

    # Game metadata
    game_id: str
    league: str
    market_type: str  # SPREAD, TOTAL, MONEYLINE
```

**Output:**

```python
@dataclass
class PredictionOutput:
    prediction_id: str
    game_id: str
    market_type: str

    # Adjusted probabilities
    adjusted_probability: float          # primary prediction
    confidence_interval_lower: float     # 90% CI lower bound
    confidence_interval_upper: float     # 90% CI upper bound

    # Comparison to simulation
    simulation_probability: float        # raw sim output for this market
    adjustment_magnitude: float          # adjusted - simulation

    # Explainability
    feature_importance: dict[str, float] # SHAP values or similar
    top_adjustments: list[dict]          # top 3 features driving the adjustment

    # Model metadata
    model_version_id: str
    calibration_score: float             # model's recent calibration error
```

### Adjustment Approach

The model does NOT predict outcomes directly. It predicts the **adjustment** to the simulation probability:

```text
adjusted_prob = simulation_prob + model_predicted_adjustment
```

This ensures the simulation output is always the baseline and the ML layer only shifts probabilities when contextual
evidence warrants it. The target variable during training is:

```text
target_adjustment = actual_outcome (0 or 1) - simulation_probability
```

This means:

- When the simulation is well-calibrated, the model learns small adjustments
- When the simulation systematically misses a contextual factor, the model learns to correct it
- The model cannot "overfit away" the simulation -- it can only add information

**Alternative considered and rejected:** Training the model to predict outcomes directly (ignoring simulation). This was
rejected because it throws away the mechanistic information in the simulation and makes the model entirely dependent on
historical patterns.

---

## 2. Feature Engineering per Sport

All features are organized into categories. Each feature includes its type, source, and which sports it applies to.

### 2.1 Simulation-Derived Features (All Sports)

These come directly from the simulation output and anchor the prediction.

| Feature                   | Type  | Description                                | Source            |
| ------------------------- | ----- | ------------------------------------------ | ----------------- |
| `sim_home_win_prob`       | float | Simulation P(home wins)                    | simulation-engine |
| `sim_away_win_prob`       | float | Simulation P(away wins)                    | simulation-engine |
| `sim_margin_mean`         | float | Mean predicted margin (home - away)        | simulation-engine |
| `sim_margin_std`          | float | Standard deviation of margin               | simulation-engine |
| `sim_total_mean`          | float | Mean predicted total score                 | simulation-engine |
| `sim_total_std`           | float | Standard deviation of total                | simulation-engine |
| `sim_spread_cover_prob`   | float | P(home covers at the current market line)  | simulation-engine |
| `sim_total_over_prob`     | float | P(over at the current market total)        | simulation-engine |
| `sim_convergence_quality` | float | 1.0 if fully converged, fraction otherwise | simulation-engine |

### 2.2 Recency Features

Performance in recent games captures form, momentum, and recent scheme changes.

| Feature                   | Type  | Sports | Description                                        |
| ------------------------- | ----- | ------ | -------------------------------------------------- |
| `home_last5_win_pct`      | float | All    | Win percentage in last 5 games                     |
| `away_last5_win_pct`      | float | All    | Win percentage in last 5 games                     |
| `home_last5_ppg`          | float | All    | Points per game in last 5                          |
| `away_last5_ppg`          | float | All    | Points per game in last 5                          |
| `home_last5_ppg_allowed`  | float | All    | Points allowed per game in last 5                  |
| `away_last5_ppg_allowed`  | float | All    | Points allowed per game in last 5                  |
| `home_last5_margin_avg`   | float | All    | Average margin of victory in last 5                |
| `away_last5_margin_avg`   | float | All    | Average margin of victory in last 5                |
| `home_last10_ats_pct`     | float | All    | Against-the-spread record in last 10 (covers / 10) |
| `away_last10_ats_pct`     | float | All    | Against-the-spread record in last 10               |
| `home_last5_total_ou_pct` | float | All    | Over rate in last 5 games                          |
| `away_last5_total_ou_pct` | float | All    | Over rate in last 5 games                          |
| `home_scoring_trend`      | float | All    | Linear slope of points scored over last 10 games   |
| `away_scoring_trend`      | float | All    | Linear slope of points scored over last 10 games   |
| `home_streak`             | int   | All    | Current win/loss streak (positive = wins)          |
| `away_streak`             | int   | All    | Current win/loss streak                            |

**Source:** `GET /teams/{id}/stats?range=last_5` and `GET /teams/{id}/stats?range=last_10` from statistics-service.

### 2.3 Situational Features

| Feature                 | Type  | Sports                     | Description                                               |
| ----------------------- | ----- | -------------------------- | --------------------------------------------------------- |
| `is_home`               | bool  | All                        | Always True for home team perspective (used for symmetry) |
| `home_rest_days`        | int   | All                        | Days since last game for home team                        |
| `away_rest_days`        | int   | All                        | Days since last game for away team                        |
| `rest_advantage`        | int   | All                        | `home_rest_days - away_rest_days`                         |
| `home_is_back_to_back`  | bool  | NBA, NCAA_BB               | Second game in consecutive days                           |
| `away_is_back_to_back`  | bool  | NBA, NCAA_BB               | Second game in consecutive days                           |
| `home_games_in_5_days`  | int   | NBA, MLB                   | Number of games played in prior 5 days                    |
| `away_games_in_5_days`  | int   | NBA, MLB                   | Number of games played in prior 5 days                    |
| `travel_distance_miles` | float | All                        | Distance between last game venue and current venue        |
| `time_zone_change`      | int   | All                        | Time zones crossed since last game                        |
| `is_neutral_site`       | bool  | All                        | Game at a neutral venue                                   |
| `altitude_ft`           | int   | NFL, NCAA_FB, NBA, NCAA_BB | Venue altitude (Denver effect)                            |
| `is_division_game`      | bool  | NFL, NBA, MLB              | Divisional/conference rival                               |
| `is_rivalry_game`       | bool  | NCAA_FB, NCAA_BB           | Named rivalry game                                        |
| `season_week`           | int   | NFL, NCAA_FB               | Week number in the season                                 |
| `season_pct_complete`   | float | All                        | Fraction of regular season completed                      |
| `is_postseason`         | bool  | All                        | Playoff/tournament game flag                              |
| `home_playoff_clinched` | bool  | All                        | Home team has clinched a playoff spot                     |
| `away_playoff_clinched` | bool  | All                        | Away team has clinched a playoff spot                     |
| `is_elimination_game`   | bool  | All                        | Loser is eliminated                                       |
| `day_of_week`           | int   | All                        | 0=Monday, 6=Sunday                                        |
| `start_hour_local`      | int   | All                        | Game start hour in local time                             |
| `is_primetime`          | bool  | NFL, NCAA_FB               | Sunday/Monday/Thursday night game                         |
| `is_short_week`         | bool  | NFL                        | Thursday game after Sunday game                           |

**Source:** `GET /games/{id}` for schedule, `GET /venues/{id}` for location, computed travel distances.

### 2.4 Weather Features (Outdoor Sports Only)

Applies to: NFL, NCAA_FB, MLB, NCAA_BSB (outdoor venues only).

| Feature              | Type        | Description                                                            |
| -------------------- | ----------- | ---------------------------------------------------------------------- |
| `temperature_f`      | float       | Temperature at game time                                               |
| `wind_speed_mph`     | float       | Wind speed                                                             |
| `wind_direction_deg` | float       | Wind direction relative to field orientation                           |
| `precipitation_prob` | float       | Probability of rain/snow                                               |
| `precipitation_type` | categorical | None, rain, snow, sleet                                                |
| `humidity_pct`       | float       | Relative humidity                                                      |
| `is_dome`            | bool        | Indoor/retractable roof closed                                         |
| `wind_x_outdoor`     | float       | `wind_speed * (1 - is_dome)` (interaction: wind only matters outdoors) |
| `cold_game`          | bool        | Temperature < 32F                                                      |
| `hot_game`           | bool        | Temperature > 90F                                                      |

**Weather impact model (for feature engineering):**

Football:

```text
passing_efficiency_adj = -0.02 * max(0, wind_speed - 15) - 0.01 * max(0, 32 - temperature)
kicking_accuracy_adj = -0.03 * max(0, wind_speed - 10)
total_adjustment = passing_efficiency_adj * 3.5 + kicking_accuracy_adj * 1.5  # points
```

Baseball:

```text
hr_distance_adj = (temperature - 72) * 0.002 + (altitude / 1000) * 0.01
run_scoring_adj = hr_distance_adj * 0.8 + wind_out_component * 0.1  # runs
```

### 2.5 Injury Features

| Feature                    | Type        | Sports        | Description                                                 |
| -------------------------- | ----------- | ------------- | ----------------------------------------------------------- |
| `home_injury_impact_score` | float       | All           | Aggregate impact of injured players (weighted by WAR/usage) |
| `away_injury_impact_score` | float       | All           | Aggregate impact of injured players                         |
| `home_qb_status`           | categorical | NFL, NCAA_FB  | starter/backup/unknown                                      |
| `away_qb_status`           | categorical | NFL, NCAA_FB  | starter/backup/unknown                                      |
| `home_qb_change_impact`    | float       | NFL, NCAA_FB  | Estimated point impact of QB change (0 if starter plays)    |
| `away_qb_change_impact`    | float       | NFL, NCAA_FB  | Estimated point impact of QB change                         |
| `home_starter_status`      | categorical | MLB, NCAA_BSB | confirmed/probable/unknown                                  |
| `away_starter_status`      | categorical | MLB, NCAA_BSB | confirmed/probable/unknown                                  |
| `home_key_players_out`     | int         | All           | Count of top-5 WAR players ruled out                        |
| `away_key_players_out`     | int         | All           | Count of top-5 WAR players ruled out                        |

**Injury impact score calculation:**

```python
def compute_injury_impact(team_injuries: list[InjuredPlayer]) -> float:
    """
    Compute aggregate injury impact as a fraction of team strength.

    Uses player's season WAR (or equivalent metric) weighted by
    probability of missing the game.
    """
    total_impact = 0.0
    for injury in team_injuries:
        # Status-based miss probability
        miss_prob = {
            'OUT': 1.0,
            'DOUBTFUL': 0.85,
            'QUESTIONABLE': 0.50,
            'PROBABLE': 0.15,
        }.get(injury.status, 0.0)

        # Impact = WAR contribution per game * miss probability
        war_per_game = injury.player_war / injury.games_played
        total_impact += war_per_game * miss_prob

    return total_impact
```

**Source:** `GET /injuries?team_id={id}` from statistics-service.

### 2.6 Market Features

| Feature                 | Type  | Sports | Description                                                |
| ----------------------- | ----- | ------ | ---------------------------------------------------------- |
| `opening_spread`        | float | All    | Opening line (home perspective)                            |
| `current_spread`        | float | All    | Current line                                               |
| `spread_movement`       | float | All    | `current_spread - opening_spread`                          |
| `spread_movement_abs`   | float | All    | `abs(spread_movement)`                                     |
| `opening_total`         | float | All    | Opening over/under                                         |
| `current_total`         | float | All    | Current over/under                                         |
| `total_movement`        | float | All    | `current_total - opening_total`                            |
| `total_movement_abs`    | float | All    | `abs(total_movement)`                                      |
| `opening_home_ml`       | int   | All    | Opening home moneyline (American odds)                     |
| `current_home_ml`       | int   | All    | Current home moneyline                                     |
| `ml_movement`           | float | All    | Implied probability change from open to current            |
| `public_bet_pct_home`   | float | All    | Fraction of public bets on home (if available)             |
| `public_money_pct_home` | float | All    | Fraction of money on home (if available)                   |
| `sharp_money_indicator` | float | All    | Money% - Bet% divergence (positive = sharp action on home) |
| `line_age_hours`        | float | All    | Hours since the line was last updated                      |
| `n_books_reporting`     | int   | All    | Number of sportsbooks with active lines                    |
| `line_consensus_std`    | float | All    | Standard deviation of lines across books                   |

**Sharp money indicator:**

```text
sharp_money = public_money_pct_home - public_bet_pct_home
```

When sharp_money > 0.10, it suggests professional bettors are on the home side (more money than ticket count implies
large bets). This is one of the strongest features for predicting line movement direction.

**Source:** `GET /lines/{game_id}` and `GET /lines/{game_id}/movement` from lines-service.

### 2.7 Historical Matchup Features

| Feature                | Type  | Sports | Description                                         |
| ---------------------- | ----- | ------ | --------------------------------------------------- |
| `h2h_home_win_pct_3yr` | float | All    | Home team's win rate in this matchup (last 3 years) |
| `h2h_avg_margin_3yr`   | float | All    | Average margin in this matchup (last 3 years)       |
| `h2h_avg_total_3yr`    | float | All    | Average total in this matchup (last 3 years)        |
| `h2h_games_count_3yr`  | int   | All    | Number of meetings (sample size signal)             |
| `h2h_home_ats_pct_3yr` | float | All    | Home ATS record in this matchup                     |
| `is_first_meeting`     | bool  | All    | No recent head-to-head history                      |

**Source:** `GET /matchup/{home_id}/{away_id}` from statistics-service.

### 2.8 Sport-Specific Features

**Football only (NFL, NCAA_FB):**

| Feature                        | Description                                    |
| ------------------------------ | ---------------------------------------------- |
| `home_off_dvoa`                | Offensive DVOA / SP+ rating                    |
| `away_def_dvoa`                | Defensive DVOA / SP+ rating                    |
| `home_third_down_conv`         | Third-down conversion rate                     |
| `away_third_down_conv_allowed` | Third-down conversion rate allowed             |
| `home_red_zone_td_pct`         | Red zone TD scoring rate                       |
| `home_turnover_margin`         | Season turnover differential                   |
| `surface_type`                 | Grass vs. turf (affects injury risk and style) |

**Basketball only (NBA, NCAA_BB):**

| Feature                   | Description                                               |
| ------------------------- | --------------------------------------------------------- |
| `pace_differential`       | Difference in team paces (affects total)                  |
| `home_three_pct_last5`    | Recent 3P% (high variance metric -- recency matters)      |
| `away_three_pct_last5`    | Recent 3P%                                                |
| `home_star_minutes_last5` | Star player average minutes in last 5 (fatigue indicator) |
| `away_star_minutes_last5` | Star player average minutes in last 5                     |
| `home_net_rating_last10`  | Net rating over last 10 games                             |
| `away_net_rating_last10`  | Net rating over last 10 games                             |

**Baseball only (MLB, NCAA_BSB):**

| Feature                          | Description                                       |
| -------------------------------- | ------------------------------------------------- |
| `home_starter_era`               | Starting pitcher ERA                              |
| `away_starter_era`               | Starting pitcher ERA                              |
| `home_starter_fip`               | Starting pitcher FIP (more predictive than ERA)   |
| `away_starter_fip`               | Starting pitcher FIP                              |
| `home_starter_k9`                | K/9 for starting pitcher                          |
| `home_starter_pitch_count_last5` | Avg pitch count in last 5 starts (workload)       |
| `home_bullpen_era_last7d`        | Bullpen ERA in last 7 days (recent usage/fatigue) |
| `away_bullpen_era_last7d`        | Bullpen ERA in last 7 days                        |
| `home_bullpen_innings_last3d`    | Bullpen innings in last 3 days (availability)     |
| `away_bullpen_innings_last3d`    | Bullpen innings in last 3 days                    |
| `umpire_k_rate_adj`              | Plate umpire's K rate relative to average         |
| `umpire_total_runs_adj`          | Plate umpire's runs/game relative to average      |
| `park_factor_runs`               | Venue run park factor                             |

---

## 3. Model Architecture

### Algorithm Selection: XGBoost

**Primary algorithm:** XGBoost gradient boosting.

**Why XGBoost over alternatives:**

| Alternative         | Why Not (Initially)                                                                                                                                                                                                                       |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Neural networks     | Insufficient training data for complex architectures. With ~2,500 NFL games/year and only 5-7 usable seasons, we have ~15,000 training examples for NFL -- far too few for deep learning. NBA (~7,000/year) is better but still marginal. |
| Random forests      | Competitive with XGBoost on accuracy but worse at capturing feature interactions and less efficient with hyperparameter tuning.                                                                                                           |
| Logistic regression | Too simple to capture nonlinear feature interactions (e.g., wind speed matters more at high altitude).                                                                                                                                    |
| LightGBM            | Viable alternative to XGBoost. Consider switching if training speed becomes a bottleneck (LightGBM is faster on large datasets).                                                                                                          |
| CatBoost            | Strong with categorical features. Consider if categorical feature handling becomes a pain point.                                                                                                                                          |

**XGBoost handles our requirements well:**

- Mixed feature types (continuous, categorical, boolean) without extensive preprocessing
- Missing values handled natively (injuries may not be reported, weather not available for dome games)
- Built-in regularization prevents overfitting on small datasets
- Feature importance is readily available (SHAP values)
- Fast inference (~1ms per prediction)

### Model Granularity Decision

**Recommendation: One model per sport, with market type as a feature.**

Rationale:

- Separate models per market type (spread, total, moneyline) would triplicate the model count to 18 (6 leagues x 3
  market types)
- Many features are shared across market types for the same game
- The model can learn market-type-specific adjustments via interaction with the `market_type` feature
- Reduces training data fragmentation (each model sees all games for that sport)

Exception: If feature importance analysis shows that market-type-specific models significantly outperform the unified
model (measured by >0.01 Brier score improvement), split into per-market models for that sport.

**Total model count:** 6 (one per league: NFL, NCAA_FB, NBA, NCAA_BB, MLB, NCAA_BSB).

### Hyperparameters

**Default XGBoost configuration:**

```python
xgb_params = {
    'objective': 'reg:squarederror',  # predicting continuous adjustment
    'eval_metric': 'rmse',
    'max_depth': 5,            # shallow trees to prevent overfitting
    'learning_rate': 0.05,     # slow learning for generalization
    'n_estimators': 500,       # tuned via early stopping
    'subsample': 0.8,          # row subsampling
    'colsample_bytree': 0.7,   # feature subsampling per tree
    'min_child_weight': 10,    # minimum samples per leaf
    'reg_alpha': 0.1,          # L1 regularization
    'reg_lambda': 1.0,         # L2 regularization
    'gamma': 0.1,              # minimum loss reduction for split
    'random_state': 42,
}
```

**Hyperparameter tuning via Optuna (Bayesian optimization):**

```python
import optuna

def objective(trial):
    params = {
        'max_depth': trial.suggest_int('max_depth', 3, 8),
        'learning_rate': trial.suggest_float('learning_rate', 0.01, 0.15, log=True),
        'n_estimators': trial.suggest_int('n_estimators', 200, 1000),
        'subsample': trial.suggest_float('subsample', 0.6, 1.0),
        'colsample_bytree': trial.suggest_float('colsample_bytree', 0.5, 1.0),
        'min_child_weight': trial.suggest_int('min_child_weight', 5, 50),
        'reg_alpha': trial.suggest_float('reg_alpha', 1e-3, 10, log=True),
        'reg_lambda': trial.suggest_float('reg_lambda', 1e-3, 10, log=True),
        'gamma': trial.suggest_float('gamma', 0, 1.0),
    }

    # Use walk-forward validation (see Training Pipeline)
    brier_scores = walk_forward_validate(params, data, n_splits=3)
    return np.mean(brier_scores)

study = optuna.create_study(direction='minimize')
study.optimize(objective, n_trials=100)
```

---

## 4. Training Pipeline

### Temporal Train/Test Split

**Critical rule: NEVER use random train/test splits.** Sports data is time-series. A model trained on 2025 data and
tested on 2023 data would have future information leakage (e.g., knowing that a player had a breakout year in 2024 would
inform predictions about their 2023 performance indirectly through feature engineering).

**Walk-forward validation:**

```text
Fold 1: Train on seasons 2020-2022, test on 2023
Fold 2: Train on seasons 2020-2023, test on 2024
Fold 3: Train on seasons 2020-2024, test on 2025
```

Each fold expands the training window and tests on the next unseen season. This mimics the real deployment scenario
where the model is trained on all available history and predicts the upcoming season.

```python
def walk_forward_validate(params, data, n_splits=3):
    """
    Walk-forward cross-validation for time-series sports data.
    """
    seasons = sorted(data['season'].unique())
    brier_scores = []

    for i in range(n_splits):
        test_season = seasons[-(i + 1)]
        train_seasons = [s for s in seasons if s < test_season]

        train_data = data[data['season'].isin(train_seasons)]
        test_data = data[data['season'] == test_season]

        X_train = train_data[feature_columns]
        y_train = train_data['target_adjustment']
        X_test = test_data[feature_columns]
        y_test = test_data['target_adjustment']

        model = xgb.XGBRegressor(**params)
        model.fit(
            X_train, y_train,
            eval_set=[(X_test, y_test)],
            early_stopping_rounds=50,
            verbose=False,
        )

        # Compute adjusted probabilities
        sim_probs = test_data['sim_probability'].values
        adjustments = model.predict(X_test)
        adjusted_probs = np.clip(sim_probs + adjustments, 0.01, 0.99)

        # Brier score
        actuals = test_data['actual_outcome'].values
        brier = np.mean((adjusted_probs - actuals) ** 2)
        brier_scores.append(brier)

    return brier_scores
```

### Feature Importance and Selection

After training, compute SHAP (SHapley Additive exPlanations) values to understand feature contributions:

```python
import shap

explainer = shap.TreeExplainer(model)
shap_values = explainer.shap_values(X_test)

# Feature importance ranking
importance = np.abs(shap_values).mean(axis=0)
feature_ranking = sorted(zip(feature_columns, importance), key=lambda x: -x[1])
```

**Feature selection criteria:**

1. Remove features with mean |SHAP| < 0.001 (no measurable contribution)
2. Remove features with >50% missing values (unless missingness itself is informative, e.g., weather features for dome
   games)
3. Remove features with correlation > 0.95 with another feature (keep the one with higher SHAP importance)
4. Monitor for features that are important in training but not in recent test data (potential overfitting signal)

### Handling Class Imbalance

For spread and moneyline markets, favorites win more often than underdogs (approximately 67% for NFL favorites ATS with
typical vig). This creates mild class imbalance.

**Approach: no resampling.** XGBoost handles mild imbalance well. Instead:

- Use calibration (Section 5) to ensure predicted probabilities are accurate regardless of base rates
- Monitor calibration separately for favorites and underdogs in reliability diagrams
- Weight recent seasons slightly higher (exponential decay with half-life of 2 seasons):

```python
sample_weights = np.exp(-0.35 * (current_season - data['season']))
model.fit(X_train, y_train, sample_weight=sample_weights)
```

### Target Variable Construction

For each market type, the target is constructed as:

**Moneyline:**

```text
target = 1 if home team won, 0 if away team won
target_adjustment = target - sim_home_win_prob
```

**Spread (home team perspective):**

```text
target = 1 if home team covered the spread, 0 otherwise
target_adjustment = target - sim_spread_cover_prob
```

**Total:**

```text
target = 1 if game went over, 0 if under
target_adjustment = target - sim_total_over_prob
```

---

## 5. Calibration

### Why Calibration Matters

A prediction model's probabilities must be **calibrated**: when we predict 60% probability, that outcome should occur
approximately 60% of the time. Miscalibrated probabilities lead to incorrect edge detection -- a model that outputs 55%
when the true probability is 52% will identify false edges.

For BookieBreaker, calibration is more important than discrimination (AUC). A model that correctly ranks game outcomes
but assigns wrong probabilities will systematically misprice edges.

### Calibration Methods

#### Method 1: Platt Scaling (logistic calibration)

Fit a logistic regression on the model's raw output probabilities using a held-out calibration set:

```text
P_calibrated = 1 / (1 + exp(-(a * P_raw + b)))
```

Where `a` and `b` are learned from the calibration set. This works well when the miscalibration is smooth and monotonic.

```python
from sklearn.calibration import CalibratedClassifierCV

# Note: we use our model's predicted probabilities as input to Platt scaling
calibrator = CalibratedClassifierCV(
    base_estimator=None,  # pre-fitted
    method='sigmoid',     # Platt scaling
    cv='prefit',
)
calibrator.fit(raw_probs.reshape(-1, 1), actuals)
calibrated_probs = calibrator.predict_proba(new_probs.reshape(-1, 1))[:, 1]
```

#### Method 2: Isotonic Regression (non-parametric calibration)

Fits a non-decreasing step function to map raw probabilities to calibrated ones. More flexible than Platt scaling but
requires more data and can overfit on small samples.

```python
from sklearn.isotonic import IsotonicRegression

calibrator = IsotonicRegression(out_of_bounds='clip')
calibrator.fit(raw_probs, actuals)
calibrated_probs = calibrator.predict(new_probs)
```

**Recommendation: Platt scaling for sports with fewer games (NFL, NCAA_FB), isotonic regression for sports with more
data (NBA, MLB).** The choice depends on calibration set size:

- < 5,000 games: Platt scaling (2 parameters, less overfitting risk)
- > = 5,000 games: Isotonic regression (more flexible, enough data to fit)

### Calibration Evaluation

**Reliability Diagram:**

Partition predictions into bins (e.g., 10 bins: 0-10%, 10-20%, ..., 90-100%). For each bin, plot the mean predicted
probability vs. the actual observed frequency. A perfectly calibrated model lies on the diagonal.

```python
def compute_calibration(predictions, actuals, n_bins=10):
    """
    Compute calibration metrics and bin data for reliability diagram.
    """
    bins = np.linspace(0, 1, n_bins + 1)
    bin_indices = np.digitize(predictions, bins) - 1
    bin_indices = np.clip(bin_indices, 0, n_bins - 1)

    bin_means_pred = []
    bin_means_actual = []
    bin_counts = []

    for i in range(n_bins):
        mask = bin_indices == i
        if mask.sum() > 0:
            bin_means_pred.append(predictions[mask].mean())
            bin_means_actual.append(actuals[mask].mean())
            bin_counts.append(mask.sum())

    # Expected Calibration Error (ECE)
    ece = sum(
        (count / len(predictions)) * abs(pred - actual)
        for pred, actual, count in zip(bin_means_pred, bin_means_actual, bin_counts)
    )

    return {
        'ece': ece,
        'bin_predictions': bin_means_pred,
        'bin_actuals': bin_means_actual,
        'bin_counts': bin_counts,
    }
```

**Brier Score Decomposition:**

The Brier score `BS = mean((p - o)^2)` decomposes into three components:

```text
BS = Reliability - Resolution + Uncertainty
```

Where:

- **Reliability** (lower is better): measures calibration error. Target: < 0.01
- **Resolution** (higher is better): measures how much predictions deviate from the base rate. Indicates discrimination
  ability.
- **Uncertainty** (constant): `base_rate * (1 - base_rate)`. Fixed for a given dataset.

```python
def brier_decomposition(predictions, actuals, n_bins=10):
    """
    Decompose Brier score into reliability, resolution, and uncertainty.
    """
    o_bar = actuals.mean()  # overall base rate
    uncertainty = o_bar * (1 - o_bar)

    bins = np.linspace(0, 1, n_bins + 1)
    bin_indices = np.digitize(predictions, bins) - 1
    bin_indices = np.clip(bin_indices, 0, n_bins - 1)

    reliability = 0.0
    resolution = 0.0
    n = len(predictions)

    for i in range(n_bins):
        mask = bin_indices == i
        n_k = mask.sum()
        if n_k == 0:
            continue
        f_k = predictions[mask].mean()  # mean forecast in bin
        o_k = actuals[mask].mean()      # observed frequency in bin

        reliability += (n_k / n) * (f_k - o_k) ** 2
        resolution += (n_k / n) * (o_k - o_bar) ** 2

    brier_score = reliability - resolution + uncertainty

    return {
        'brier_score': brier_score,
        'reliability': reliability,
        'resolution': resolution,
        'uncertainty': uncertainty,
    }
```

### Calibration Targets

| Metric                           | Target                              | Failure Threshold |
| -------------------------------- | ----------------------------------- | ----------------- |
| Expected Calibration Error (ECE) | < 0.03                              | > 0.05            |
| Brier Score (overall)            | < 0.24 (spread/ML), < 0.23 (totals) | > 0.26            |
| Brier Reliability                | < 0.01                              | > 0.02            |
| Max bin deviation                | < 0.08                              | > 0.12            |

If any metric exceeds its failure threshold, the model should not be promoted to active status.

---

## 6. Model Versioning and A/B Testing

### Version Naming Convention

Model versions are identified by a composite key:

```text
{league}_{market_scope}_{timestamp}_{metrics_hash}
```

Example: `NFL_unified_20260315_a3f2b1c4`

Where:

- `league`: NFL, NCAA_FB, NBA, NCAA_BB, MLB, NCAA_BSB
- `market_scope`: "unified" (all market types) or specific (e.g., "spread")
- `timestamp`: YYYYMMDD of training completion
- `metrics_hash`: first 8 chars of SHA-256 of the model's evaluation metrics JSON (ensures uniqueness even if retrained
  on the same day)

### Model Registry Schema

```python
@dataclass
class ModelVersion:
    id: str                       # UUID
    version_name: str             # e.g., "NFL_unified_20260315_a3f2b1c4"
    league: str
    market_scope: str
    algorithm: str                # "xgboost", "lightgbm", etc.
    hyperparameters: dict         # full hyperparameter dict
    training_seasons: list[int]   # seasons used for training
    training_samples: int         # number of training examples
    feature_columns: list[str]    # ordered list of features used
    evaluation_metrics: dict      # Brier score, ECE, resolution, etc.
    calibration_method: str       # "platt" or "isotonic"
    is_active: bool               # currently serving predictions
    is_shadow: bool               # running in shadow mode alongside active
    created_at: datetime
    promoted_at: datetime | None  # when moved from shadow to active
    retired_at: datetime | None
    artifact_path: str            # path to serialized model file
```

### Shadow Mode (A/B Testing)

New models run in **shadow mode** before promotion:

1. **Training:** New model is trained on expanded data (new season, new features, tuned hyperparameters).
2. **Shadow deployment:** New model runs alongside the active model for every prediction request. Both models produce
   predictions; only the active model's predictions are used for edge detection.
3. **Evaluation period:** Shadow model accumulates predictions over a minimum sample:
   - NFL/NCAA_FB: 50 games (approximately 3-4 weeks)
   - NBA/NCAA_BB: 100 games (approximately 1-2 weeks)
   - MLB/NCAA_BSB: 150 games (approximately 1 week)
4. **Comparison:** Compare shadow vs. active model on:
   - Brier score (lower is better)
   - ECE (lower is better)
   - CLV of recommended edges (positive is better)
5. **Promotion criteria:** Shadow model is promoted if:
   - Brier score is lower by at least 0.005 (statistically significant improvement)
   - ECE does not increase by more than 0.005
   - No calibration bin deviates by more than 0.10
6. **Automatic retirement:** If the shadow model fails to meet promotion criteria after 2x the minimum evaluation
   period, it is retired and logged.

### Automatic Retraining Triggers

| Trigger                    | Condition                                                           | Action                                                        |
| -------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------- |
| Scheduled                  | Monthly (1st of each month during active seasons)                   | Retrain with all available data                               |
| New season start           | First game of a new season detected                                 | Retrain with previous season added                            |
| Performance degradation    | Rolling 100-game Brier score > active model's training Brier + 0.02 | Trigger retraining with alert                                 |
| Significant roster changes | Trade deadline, free agency period, transfer portal window          | Retrain with updated roster features                          |
| Rule change                | Manual trigger by operator                                          | Retrain; consider discarding or downweighting pre-change data |

### Retraining Pipeline

```python
def retrain_model(league: str, trigger: str) -> ModelVersion:
    """
    Full retraining pipeline.

    Steps:
    1. Fetch all historical data for the league
    2. Compute features (feature engineering pipeline)
    3. Tune hyperparameters via Optuna (if trigger is 'scheduled' or 'new_season')
    4. Train final model on all data
    5. Calibrate on most recent season
    6. Evaluate via walk-forward validation
    7. Register new ModelVersion
    8. Deploy in shadow mode
    """
    # 1. Data collection
    data = fetch_historical_data(league, min_season=2020)

    # 2. Feature engineering
    features = compute_features(data, league)

    # 3. Hyperparameter tuning (only on scheduled retrains)
    if trigger in ('scheduled', 'new_season'):
        params = tune_hyperparameters(features, n_trials=100)
    else:
        params = get_active_model(league).hyperparameters

    # 4. Train
    model = train_model(features, params)

    # 5. Calibrate
    calibrator = calibrate_model(model, features, league)

    # 6. Evaluate
    metrics = walk_forward_evaluate(model, calibrator, features)

    # 7. Register
    version = register_model(model, calibrator, metrics, league, trigger)

    # 8. Shadow deploy
    deploy_shadow(version)

    return version
```
