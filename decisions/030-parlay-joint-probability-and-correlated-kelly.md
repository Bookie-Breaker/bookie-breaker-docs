# ADR-030: Parlay Joint Probability and Correlated Kelly

## Status

Accepted

## Context

Phase 7 Wave 1 implements edge-detection.md §5 (parlay correlation). The §5 spec provides a first-order
approximation for multi-leg joint probability but explicitly warns it breaks down for `rho > 0.3` or 3+ correlated
legs — exactly the same-game-parlay case that motivates the feature (soccer result + total during the World Cup).
The spec's own remedy is "use Monte Carlo simulation to estimate the joint probability by drawing from the
simulation output distributions directly."

Two design questions follow:

1. **Where does the joint structure come from, and in what form?** The simulation-engine already draws
   index-aligned per-iteration samples (score/margin/total arrays) — the joint structure exists but only 1-D
   marginals leave the service.
2. **Whose marginals feed the joint?** The simulation's marginals are uncalibrated; the prediction-engine's
   calibrated probabilities are the system's best per-leg estimates (ADR-002/014). A raw simulation joint would
   silently discard calibration.

## Decision

**1. The simulation-engine exposes a correlation artifact over a canonical leg vocabulary.**
`GET /api/v1/sim/simulations/{run_id}/correlations[?legs=...]` returns, for legs named
`MONEYLINE:HOME|AWAY|DRAW`, `SPREAD:HOME|AWAY:{line}`, `TOTAL:OVER|UNDER:{line}` (lines `%g`-formatted; pushes
count as losses in leg vectors):

- empirical `marginals` and the pairwise phi/Pearson `matrix` over the boolean leg vectors,
- the empirical `joint_probability` of a requested leg set (the AND across aligned iterations — §5's Monte-Carlo
  path, exact up to MC error, no approximation),
- soccer/hockey additionally expose their analytic `joint_goal_grid` (the Dixon-Coles PMF the plugin already
  builds).

The artifact is computed at run time from the in-memory draws and cached (`sim:correlations:{game_id}`) as a
bit-packed boolean leg matrix, so arbitrary-subset joints remain answerable after the raw arrays are gone. The
leg-key vocabulary is a cross-service contract (agent ↔ simulation-engine).

**2. The agent combines calibrated marginals with the simulation's joint structure ("simulation-scaled").**
For a same-game leg group with a simulation run available:

```text
joint = sim_joint × Π(calibrated_i / sim_marginal_i)
```

clamped to the Fréchet bounds `[max(0, Σp − (n−1)), min(pᵢ)]`. This keeps the MC joint dependence structure while
letting the calibrated per-leg probabilities set the level. Fallbacks, in order: §5's first-order approximation
with the doc's correlation-prior table when no simulation run exists (`method="prior_first_order"`); independence
across different games (cross-game correlations like shared weather are out of scope for v1). Single-league
parlays only in v1.

**3. Position sizing uses Kelly on the joint outcome.** `correlated_kelly(joint_probability,
combined_decimal_odds)` with the same 1/4-Kelly multiplier and caps as single bets (edge-detection.md §3), then
the existing simultaneous-exposure scaling. Combined decimal odds = product of leg decimals (or the book's offered
SGP price when supplied — the comparison of the two is the mispricing signal, `correlation_edge`).

**4. §5's first-order threshold becomes a fallback guard, not the primary path.** Because the simulation-scaled
path is the default for same-game groups, the `rho > 0.3` / 3+-legs breakdown zone of the first-order formula is
only reachable in the prior-fallback path, where results carry the lower-confidence `method` marker. The Phase 7
plan's Wave 4 "MC-joint upgrade" is therefore substantially delivered by this ADR; Wave 4 revisits only
player-prop legs (which need Wave 3's player distributions in the leg vocabulary).

## Consequences

### Positive

- Same-game joints are exact-up-to-MC-error rather than first-order-approximate, precisely where the money is
  (WC soccer SGPs)
- Calibration is preserved: the prediction-engine's Platt/conformal work keeps setting probability levels
- The packed-matrix cache answers any leg-subset joint without re-simulation, and the artifact is small
  (~40 legs × 10k iterations ≈ 50 KB packed)
- The pairwise matrix is exactly §5's `estimate_correlation` computed once where the draws live

### Negative

- The scaling identity is a heuristic: it is exact when calibration is a monotone per-leg adjustment, but can
  distort the dependence structure for large calibration shifts (mitigated by Fréchet clamping and by reporting
  `independent_probability` alongside for sanity checks)
- Correlation artifacts add a per-run compute + cache cost (bounded: one corrcoef over a boolean matrix)
- Cross-game correlation (weather systems, divisional effects) is knowingly ignored in v1

### Neutral

- The leg vocabulary is intentionally minimal (team markets); player-prop legs extend it in Wave 3 with
  `PLAYER_PROP:{player}:{stat}:{side}:{line}` keys
- The doc's prior table remains useful as the no-simulation fallback and for cross-checking empirical values
