# ADR-018: Football Simulation Granularity

## Status
Accepted

## Context
Phase 6 adds NFL and NCAA Football simulation plugins to the simulation-engine. We need to decide the simulation granularity for football.

Options considered:

1. **Drive-based simulation:** Simulate each possession as a unit. Model drive outcomes (touchdown, field goal, punt, turnover) using team-level efficiency metrics. Faster, simpler, sufficient for spread/total/moneyline predictions.
2. **Play-level simulation:** Simulate individual plays within each drive. Model play type, yards gained, turnovers at the play level. More granular, enables player prop predictions, but significantly more complex and slower.

## Decision
Start with **drive-based simulation** for Phase 6. Add play-level granularity in Phase 7 when player prop modeling requires it.

Drive-based simulation is sufficient for the core bet types (spread, total, moneyline) that Phase 6 targets. It models:
- Possessions per game (based on team pace / time of possession)
- Drive outcome probabilities (TD, FG, punt, turnover, end of half) using team offensive/defensive efficiency
- Scoring distribution from drive outcomes
- Home field advantage adjustments

This produces score distributions, margin distributions, and total distributions — everything the prediction-engine needs for edge detection on the primary bet types.

Play-level simulation is deferred to Phase 7, where it's needed for player prop predictions (passing yards, rushing yards, receiving yards, etc.). At that point, the drive-based framework can be extended to decompose drives into individual plays.

## Consequences

### Positive
- Drive-based simulation is 10-50x faster than play-level, enabling full-slate simulation in seconds
- Simpler to implement and validate — fewer parameters, easier to calibrate against historical results
- Sufficient for all Phase 6 bet types (spread, total, moneyline)
- Provides a working foundation that Phase 7 extends rather than replaces

### Negative
- Cannot produce player stat distributions (needed for Phase 7 player props) without the play-level extension
- Drive-level abstraction loses some game dynamics (e.g., clock management, two-minute drill, garbage time scoring patterns)
- May underperform play-level models for games with extreme pace differentials

### Neutral
- The simulation-engine plugin architecture (ADR-001) already separates the simulation interface from the sport-specific implementation — switching from drive-based to play-based is an internal change to the football plugin, not a framework change
- NCAA Football can reuse the same drive-based model with adjusted parameters (college game rules: different clock rules, overtime format)
- nfl_data_py provides drive-level summary data that directly feeds drive-based simulation parameters
