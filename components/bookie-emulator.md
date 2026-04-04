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
- Does NOT expose MCP tools directly. The bookie-emulator stays as a REST API; its capabilities are exposed to MCP clients through the centralized [mcp-server](./mcp-server.md), which wraps the bookie-emulator API as MCP tools (`place_bet`, `get_bets`, `get_performance`). This avoids coupling transport concerns into a domain service and prevents context bloat in LLM interactions.

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

---

## Requirements

### Functional Requirements

- **FR-001:** Accept paper bet placement requests from the agent containing game_id, bet type, selection, predicted probability, edge size, and stake sizing. Validate all fields before persisting.
- **FR-002:** At time of bet placement, capture the exact current line/odds from lines-service and store as placement odds on the PaperBet record.
- **FR-003:** Support idempotent bet placement via `X-Idempotency-Key` header to prevent duplicate bets from retried requests. Keys expire after 24 hours.
- **FR-004:** Persist all open and historical paper bets with full context: game, bet type, side, line value, odds at placement, stake (units + dollars), predicted probability, edge size, and reasoning.
- **FR-005:** Grade bets automatically when a `game.completed` event is received: determine win/loss/push by comparing the bet's terms against the final game score from statistics-service.
- **FR-006:** As a fallback, periodically poll statistics-service (every 30 minutes) for final scores on all open bets to catch any missed `game.completed` events.
- **FR-007:** After grading a bet, capture the closing line from lines-service and compute Closing Line Value (CLV) as the difference in implied probability between placement odds and closing odds.
- **FR-008:** Calculate profit/loss for each graded bet: winning bet P/L = `stake * (odds_decimal - 1)`, losing bet P/L = `-stake`, push P/L = `0`.
- **FR-009:** Compute and serve aggregate performance metrics: ROI, win rate, average CLV, units won/lost, longest win/loss streaks, and calibration data. Support breakdowns by sport, league, bet type, and time window (daily, weekly, monthly, all-time).
- **FR-010:** Take BankrollSnapshot after each bet is graded and at end-of-day, capturing cumulative bankroll state for performance trend charting.
- **FR-011:** Enforce configurable bankroll management rules: maximum bet size (in units), maximum daily exposure, and Kelly criterion fractional sizing (if enabled).
- **FR-012:** Serve the full bet ledger with filtering by league, bet type, date range, result, and edge size.
- **FR-013:** Produce calibration data: group predictions by probability bucket (e.g., 50-55%, 55-60%) and report actual win rate per bucket to evaluate prediction accuracy.

### Non-Functional Requirements

- **Latency:** Bet placement (including odds capture from lines-service): < 2 seconds. Bet grading (including score fetch and CLV calculation): < 5 seconds per bet. Performance metric queries: < 500ms. Bet ledger queries with filters: < 1 second.
- **Throughput:** Handle up to 50 bet placements/day (typical: 5-20). Grade up to 50 bets/day. Serve up to 50 API requests/second for performance and ledger queries from agent and interfaces.
- **Availability:** 99.5% uptime. Graceful degradation: if lines-service is unavailable at placement time, queue the bet and capture odds as soon as lines-service recovers (within a configurable timeout, e.g., 30 seconds, after which the placement is rejected). If statistics-service is unavailable for grading, retry with exponential backoff.
- **Storage:** Each PaperBet record is ~1-2 KB. Each BetGrade is ~500 bytes. Each BankrollSnapshot is ~500 bytes. Estimated 5,000-20,000 bets/year. Total: ~50-100 MB/year. Growth is modest.

### Data Ownership

This service is the source of truth for:
- **PaperBet** -- all paper bets placed, with placement odds, stake, predicted probability, edge, result, P/L, and CLV.
- **BetGrade** -- detailed grading records with actual scores and result determination logic.
- **BankrollSnapshot** -- point-in-time bankroll state for performance tracking over time.

### APIs Exposed

| Method + Path | Description | Key Query Parameters | Consumers |
|---|---|---|---|
| `POST /api/v1/bets` | Place a new paper bet | Body: game_id, market_type, selection, side, predicted_probability, edge_percentage, stake. Header: X-Idempotency-Key | agent |
| `GET /api/v1/bets` | List paper bets (bet ledger) | `league`, `market_type`, `result`, `date_from`, `date_to`, `min_edge`, `limit`, `offset` | agent, CLI, UI, MCP |
| `GET /api/v1/bets/{bet_id}` | Get a specific bet with grade details | -- | agent, CLI, UI, MCP |
| `GET /api/v1/bets/open` | List currently open (ungraded) bets | `league` | agent, CLI, UI, MCP |
| `GET /api/v1/performance` | Get aggregate performance metrics | `league`, `market_type`, `date_from`, `date_to`, `window` (daily/weekly/monthly/all-time) | agent, CLI, UI, MCP |
| `GET /api/v1/performance/calibration` | Get calibration curve data | `league`, `market_type` | agent, CLI, UI, MCP |
| `GET /api/v1/performance/bankroll` | Get bankroll history for charting | `date_from`, `date_to`, `interval` | UI, CLI |
| `GET /api/v1/performance/breakdown` | Get performance breakdown by sport/league/bet type | `group_by` (league, market_type, sportsbook) | agent, CLI, UI, MCP |

### APIs Consumed

| Service | Endpoint | Purpose |
|---|---|---|
| lines-service | `GET /api/v1/lines/snapshot` | Capture exact current odds at bet placement time |
| lines-service | `GET /api/v1/lines/{game_id}/closing` | Get closing lines for CLV calculation after game completion |
| statistics-service | `GET /api/v1/games/{game_id}/result` | Get final game score for bet grading |

### Events Published

None. The bookie-emulator does not publish events. Its data is consumed via synchronous API calls.

### Events Subscribed

| Event | Channel | Purpose |
|---|---|---|
| `game.completed` | `events:game.completed` | Triggers automated bet grading for any open bets on the completed game. |

### Storage Requirements

- **Database:** PostgreSQL.
- **Key tables:**
  - `paper_bets` -- all paper bets, open and graded (~5K-20K rows/year).
  - `bet_grades` -- grading details per graded bet (~5K-20K rows/year).
  - `bankroll_snapshots` -- point-in-time bankroll state (~1K-5K rows/year).
  - `idempotency_keys` -- for duplicate bet prevention (~5K-20K rows/year, with 24h TTL cleanup).
  - `bankroll_config` -- bankroll management settings (unit size, max bet, Kelly fraction). ~1 row, updated occasionally.
- **Estimated row counts:** Year 1: ~10K-40K rows total. Year 3: ~30K-120K rows.
- **Growth:** ~20-100 new rows/day (bets + grades + snapshots).
- **Indexing priorities:**
  1. `paper_bets(game_id, result)` -- open bet lookup for grading.
  2. `paper_bets(placed_at DESC)` -- chronological bet ledger.
  3. `paper_bets(market_type, result)` -- performance breakdown queries.
  4. `bankroll_snapshots(timestamp)` -- time-series queries for charting.
  5. `idempotency_keys(key, created_at)` -- duplicate detection with TTL cleanup.
- **Redis:** Used for idempotency key cache (24h TTL), recently graded bet cache (1h TTL), and event deduplication (1h TTL).
