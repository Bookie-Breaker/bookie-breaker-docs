# ADR-014: Probability Calibration and Confidence Estimation

## Status

Accepted

## Context

The prediction-engine produces calibrated probabilities for spread, total, and moneyline outcomes. XGBoost raw outputs
are not inherently calibrated — a model that outputs 0.7 does not necessarily mean the event occurs 70% of the time. A
calibration layer is needed to convert raw scores to true probabilities.

Two decisions are bundled here because they are tightly coupled:

**Calibration method:**

1. **Platt scaling:** Fits a 2-parameter sigmoid to map raw outputs to probabilities. Works well with small datasets.
   Assumes a sigmoid relationship.
2. **Isotonic regression:** Non-parametric, fits a non-decreasing step function. More flexible but requires more data to
   avoid overfitting.

**Confidence interval estimation:**

1. **Bootstrap resampling:** Re-train or re-predict on resampled data to estimate prediction variance. Model-agnostic.
   Computationally expensive.
2. **Model-based (conformal prediction):** Uses held-out calibration set to produce prediction intervals with coverage
   guarantees. More principled, less compute at inference time.

## Decision

**Calibration:** Start with **Platt scaling**, switch to **isotonic regression** per-sport once sufficient data
accumulates.

The algorithms doc (`algorithms/prediction-models.md`) recommends Platt scaling for sports with fewer than ~5,000
historical games and isotonic regression for larger datasets. NBA has ~1,300 regular season games per year, so ~5
seasons of data crosses the threshold. NFL has only ~272 games per season, so Platt scaling may remain appropriate
longer.

The calibration method is stored as model metadata — each model version records which calibration was used. This allows
per-sport, per-model-version flexibility without a global switch.

**Confidence intervals:** Use **conformal prediction** (split conformal method).

Conformal prediction provides distribution-free coverage guarantees: a 90% prediction interval contains the true outcome
at least 90% of the time, regardless of the underlying model. It requires only a held-out calibration set (not
retraining), making it fast at inference time. The split conformal method is straightforward to implement and
well-suited to production serving.

## Consequences

### Positive

- Platt scaling is simple to implement and robust with limited training data (critical for early phases)
- Per-sport calibration method selection adapts to data availability as it grows
- Conformal prediction provides rigorous coverage guarantees without distributional assumptions
- Both methods are well-understood in the ML literature with mature scikit-learn implementations

### Negative

- Platt scaling assumes a sigmoid relationship between raw scores and true probabilities — may under-fit for some sports
- Switching calibration methods over time means prediction intervals from different model versions are not directly
  comparable
- Conformal prediction intervals can be wide when the model is uncertain, which may reduce their usefulness for edge
  detection

### Neutral

- Model versioning (already planned) tracks which calibration method produced each prediction
- Calibration quality is measurable via calibration curves — can monitor and switch methods based on evidence
- Both Platt scaling and isotonic regression are available in scikit-learn (`CalibratedClassifierCV`)
