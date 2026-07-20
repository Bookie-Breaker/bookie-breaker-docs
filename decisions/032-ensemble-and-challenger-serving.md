# ADR-032: Ensemble Models and Champion/Challenger Serving

## Status

Accepted

## Context

Phase 7 Wave 4 adds the roadmap's "Advanced ML" items: ensemble methods, a model A/B-testing framework, and an
automated retraining pipeline. Three constraints shape the design:

1. **The artifact store is pickle-free by decision** (`artifact.py` serializes XGBoost natively as `.ubj` plus JSON
   sidecars). A scikit-learn random forest or a neural net would break that contract.
2. **The database enforces one active model per (sport, market)** via a partial unique index — correct for simple
   serving, incompatible with running two models side by side.
3. **Real money is not at stake, but paper-trading integrity is**: the bankroll and performance analytics are the
   product. A challenger model must be able to prove itself without contaminating the ledger.

## Decision

**1. Ensembles are XGBoost-only blends.** `EnsembleAdjustmentModel` holds ordered members with non-negative
weights summing to 1: the existing gradient-boosted adjustment model plus an **XGBoost-native random forest**
(`num_parallel_tree` ≥ 50, one boosting round, row/column subsampling) — both serialize to `.ubj`, keeping the
pickle-free contract. Blend weights are fit on the calibration split by minimizing Brier over a simplex grid;
Platt calibration and conformal intervals apply to the blended output. Artifacts without a `blend.json` load as
single-model bundles, so every deployed artifact remains valid. Neural-net members are deliberately deferred
(serialization and overfitting cost outweigh the marginal gain at our sample sizes — the roadmap's "and/or"
resolves to "or").

**2. Serving gains a `role` dimension: champion / challenger / shadow.** The partial unique index becomes
`(sport, model_type, role) WHERE is_active` — one active model _per role_. Consumers see only the champion; the
registry serves it by default and the public API behavior is unchanged.

**3. Shadow scoring is the default comparison mode.** When an active challenger exists, every prediction request
also scores the challenger and persists its row flagged `is_shadow=true`; all read paths (latest, edges, agent
consumption) filter shadows out. Zero bankroll exposure, full paired comparison data. An optional deterministic
traffic split (`AB_SPLIT_PCT`, default 0) can promote the challenger to primary for a hashed fraction of games —
an explicit operator action, never a default.

**4. Retraining produces challengers, never champions.** `POST /models/retrain` (a stub since Phase 2) becomes a
background job: assemble training rows by joining stored feature vectors with **outcomes settled from
statistics-service finals** (regulation scores for soccer per ADR-027; pushes dropped; player-prop rows excluded
from v1 assembly — a tracked deferral), walk-forward retrain, register as an active challenger. Promotion is a
separate explicit call gated by criteria — the challenger must beat the champion on Brier **and** log-loss over a
minimum sample of graded shadow pairs with calibration no worse — or an operator `force`. `GET
/models/experiments/{sport}/{market}` exposes the running comparison and a `promotion_ready` flag.

## Consequences

### Positive

- A/B evidence accumulates continuously at zero bankroll risk; promotion is a deliberate, criteria-gated act
- The pickle-free contract survives; ensemble artifacts remain portable single-directory bundles
- Retraining finally closes the loop the roadmap promised in Phase 2 ("automated retraining pipeline") using data
  the system already stores (feature vectors + finals)
- Backward compatible end to end: existing artifacts load, existing consumers see identical responses

### Negative

- Shadow scoring roughly doubles prediction-row volume while a challenger is active (bounded: one challenger per
  (sport, market), rows flagged and filterable)
- Outcome assembly depends on statistics-service finals availability; sparse offseason data limits how fast a
  challenger can qualify (the promotion sample gate makes this explicit rather than silent)
- The simplex-grid blend fit is crude; acceptable at two members, revisit if the member set grows

### Neutral

- Player-prop retraining assembly is deferred until graded prop history accumulates (box-score-settled outcomes
  exist since Wave 3; the join is future work, tracked in the roadmap checklist)
- kagent (ADR-023) was re-evaluated for this wave and remains deferred: the system is still Compose-based and
  this ADR's serving needs are met in-process
