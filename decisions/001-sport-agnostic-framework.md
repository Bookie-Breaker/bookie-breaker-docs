# ADR-001: Sport-Agnostic Framework with 6 Initial Leagues

## Status
Accepted

## Context
BookieBreaker needs to support multiple sports. We had three options: build for a single sport first, build for multiple sports from day one with sport-specific code, or build a sport-agnostic framework that can be configured per sport.

Each sport has fundamentally different simulation and statistical characteristics (discrete plays in football vs. continuous flow in basketball vs. pitcher-batter matchups in baseball), but the overall pipeline — ingest data, simulate outcomes, predict probabilities, detect edges — is the same.

## Decision
Build a sport-agnostic framework that supports 6 leagues from the start:
- **Football:** NFL, NCAA Football
- **Basketball:** NBA, NCAA Basketball
- **Baseball:** MLB, NCAA Baseball

The framework defines sport-neutral interfaces (e.g., `GameSimulator`, `FeatureExtractor`, `LineNormalizer`) with sport-specific implementations plugged in. Shared infrastructure (ingestion scheduling, edge detection math, paper trading) remains sport-agnostic.

## Consequences

### Positive
- Forced abstraction prevents sport-specific assumptions from leaking into core infrastructure
- Adding a new sport (NHL, soccer, etc.) becomes a matter of implementing the sport plugin interface
- Pro and college variants of the same sport share significant modeling logic

### Negative
- More upfront design work to get the abstractions right before any sport works end-to-end
- Risk of over-abstraction — some sport-specific concerns may not fit cleanly into a generic interface
- 6 leagues means 6x the data source integrations from the start

### Neutral
- Pro/college pairs (NFL/NCAAF, NBA/NCAAB, MLB/NCAABB) will share most simulation logic but differ in data quality and availability
