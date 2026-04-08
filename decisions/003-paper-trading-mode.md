# ADR-003: Paper Trading Mode for Validation

## Status

Accepted

## Context

Before risking real money, the system needs a way to validate its predictions and edge-detection accuracy. Options considered:

1. **Historical backtesting** — Replay past seasons against historical lines to measure theoretical performance
2. **Paper trading** — Simulate placing bets in real-time against live lines without real money, tracking results as games complete
3. **Both** — Support both modes

## Decision

The bookie-emulator service implements paper trading with accuracy tracking. It does not perform historical backtesting.

Paper trading means:

- When the system detects an edge, the emulator can "place" a virtual bet at the current line/odds
- Virtual bets are tracked with all relevant metadata (line, odds, stake, timestamp, reasoning)
- When games complete, bets are graded against actual results
- Performance metrics are calculated: ROI, win rate, CLV (closing line value), calibration curves, Brier scores

## Consequences

### Positive

- Tests the full live pipeline end-to-end, including data freshness and latency
- Performance reflects real conditions (you bet the line that was actually available)
- No risk of overfitting to historical data (a common backtesting trap)
- Builds confidence in the system before any real money is involved

### Negative

- Requires the system to be running live to collect data — no way to fast-forward through historical seasons
- Slower feedback loop than backtesting (must wait for games to be played)
- Cannot evaluate how the system would have performed in past seasons

### Neutral

- Backtesting could be added later as a separate concern if needed, but is not part of the initial scope
