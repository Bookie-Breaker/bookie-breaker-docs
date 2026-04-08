# ADR-002: Hybrid Prediction Approach (Simulation + ML)

## Status

Accepted

## Context

The prediction engine needs to generate probabilities for game outcomes and betting markets. Three approaches were
considered:

1. **Pure statistical/ML models** — Train models directly on historical data to predict outcomes
2. **Pure Monte Carlo simulation** — Simulate games thousands of times to derive outcome distributions
3. **Hybrid** — Simulation generates base probabilities, ML models adjust for additional contextual factors

Pure ML struggles to capture game dynamics and produces opaque predictions. Pure simulation struggles to incorporate
soft factors (coaching changes, motivation, travel fatigue) and requires strong parametric assumptions.

## Decision

Use a hybrid approach:

1. **Simulation engine** runs Monte Carlo simulations to generate base probability distributions for game outcomes
   (scores, spreads, totals)
2. **Prediction engine** applies ML models that adjust simulation outputs based on contextual factors the simulation
   doesn't capture (injuries, weather, rest days, public betting percentages, line movement patterns, etc.)

The simulation engine is the intellectual core. The prediction engine is a calibration and adjustment layer on top.

## Consequences

### Positive

- Simulations provide interpretable, mechanistic predictions (you can trace _why_ a probability is what it is)
- ML layer captures patterns that are hard to simulate explicitly
- Each component can be evaluated and improved independently
- Simulation distributions are useful beyond point predictions (e.g., probability of covering various alternate spreads)

### Negative

- Two complex systems to build and maintain instead of one
- Need to carefully design the interface between simulation output and ML input
- Risk of ML layer overfitting to simulation biases rather than correcting them

### Neutral

- Requires clear ownership boundaries between simulation-engine and prediction-engine repos
