# bookie-breaker-bookie-emulator

## Purpose

Paper trading system that virtually places bets at current market lines/odds when the system detects an edge, tracks those bets through game completion, grades them, and computes performance metrics. Provides a risk-free way to validate the system's edge-detection accuracy before real money is involved.

## Responsibilities

- Accepts virtual bet placements with the exact line/odds at time of placement, stake sizing, and bet type.
- Persists all open and historical paper bets.
- Grades bets when games complete (win/loss/push) using final game results.
- Computes and serves performance metrics: ROI, win rate, closing line value (CLV), calibration curves, units won/lost, streak data, and breakdowns by sport/league/bet-type.
- Tracks performance over configurable time windows (daily, weekly, monthly, all-time).
- Enforces bankroll management rules (e.g., max bet size, Kelly criterion sizing) if configured.

## Non-Responsibilities

- Does NOT detect edges or decide which bets to place. The agent decides; this service executes.
- Does NOT perform historical backtesting against past games. It only tracks forward-looking paper bets placed at the time of detection.
- Does NOT source game results itself. It receives final scores from the statistics-service.
- Does NOT interact with any real sportsbook or handle real money.
- Does NOT generate analysis or explanations of performance. The agent handles interpretation.

## Inputs

| Source | Data | Mechanism |
|---|---|---|
| agent | Paper bet placement requests (game, bet type, line, odds, stake, edge size, predicted probability) | API call |
| statistics-service | Final game results for grading open bets | API call or event |
| lines-service | Current lines/odds at time of bet placement (for CLV tracking, captured at placement) | API call |

## Outputs

| Destination | Data | Mechanism |
|---|---|---|
| agent | Bet confirmations, current open bets, graded bet results | API response |
| agent / CLI / UI | Performance metrics (ROI, win rate, CLV, calibration) | API response |
| agent / CLI / UI | Historical bet ledger with filters | API response |

## Dependencies

- **statistics-service** -- provides final game scores to grade bets
- **lines-service** -- provides closing lines for CLV calculation

## Dependents

- **bookie-breaker-agent** -- places paper bets and queries performance
- **bookie-breaker-cli** -- displays bet history and performance directly
- **bookie-breaker-ui** -- renders performance dashboards and bet ledger
- **bookie-breaker-mcp-server** -- exposes paper trading capabilities as MCP tools
