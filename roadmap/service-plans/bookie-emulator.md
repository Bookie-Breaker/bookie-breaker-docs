# PLANNING: bookie-emulator

## Service

- **Name:** bookie-emulator
- **Language:** Python 3.12+
- **Framework:** FastAPI

## Implementation Phase

Phase 3 (First Interface & Paper Trading)

## Purpose

Paper trading system that validates the prediction pipeline's effectiveness. Accepts virtual bet placements, grades bets against final game results, computes Closing Line Value (CLV), and maintains aggregate performance metrics (ROI, win rate, calibration).

## Ordered Task List

- [ ] Initialize Python project: `pyproject.toml` with uv, `src/` layout, FastAPI app scaffold
- [ ] Set up FastAPI server with uvicorn, CORS middleware, request logging
- [ ] Implement health check endpoint (`GET /healthz`)
- [ ] Implement Postgres connection (asyncpg, per [ADR-013](../../decisions/013-python-postgres-driver.md)) for `emulator` schema
- [ ] Design database schema:
  - [ ] `paper_bets` table: id, game_id, sport, bet_type, side, odds_at_placement, implied_prob, predicted_prob, edge_size, stake, status (OPEN/WON/LOST/PUSH), profit_loss, placed_at, graded_at
  - [ ] `bankroll_snapshots` table: date, balance, total_wagered, total_returned
- [ ] Implement database migrations (Alembic, per [ADR-019](../../decisions/019-database-migration-tooling.md))
- [ ] Implement bet placement:
  - [ ] Accept bet type, game, side, current odds, stake, predicted probability, edge size
  - [ ] Validate bet parameters
  - [ ] Persist with status OPEN
- [ ] Implement bet grading:
  - [ ] Subscribe to `game.completed` Redis events
  - [ ] Fetch final scores from statistics-service
  - [ ] Grade bets as WON, LOST, or PUSH based on final scores and bet type/line
  - [ ] Handle push scenarios correctly (return of stake)
  - [ ] Polling fallback: periodically check for completed games with ungraded bets
- [ ] Implement CLV computation: compare placement odds to closing odds (fetched from lines-service)
- [ ] Implement performance analytics:
  - [ ] Overall ROI: total profit / total wagered
  - [ ] Win rate: wins / (wins + losses)
  - [ ] Units won/lost
  - [ ] CLV average
  - [ ] Breakdown by sport, bet type, time window
- [ ] Implement bet ledger: query bets with filters (sport, bet type, date range, outcome) and sorting
- [ ] Build REST API:
  - [ ] `POST /api/v1/bets` -- place a paper bet
  - [ ] `GET /api/v1/bets` -- list bets with filters and sorting
  - [ ] `GET /api/v1/bets/{id}` -- single bet detail
  - [ ] `GET /api/v1/performance` -- aggregate performance metrics
  - [ ] `GET /api/v1/performance/history` -- daily performance snapshots for charting
- [ ] Write unit tests for bet grading logic (all bet types, edge cases like pushes)
- [ ] Write integration tests for full bet lifecycle
- [ ] Create Dockerfile and integrate into Docker Compose
- [ ] Add `.env.example`

**Phase 7 additions:**

- [ ] Implement player prop bet grading (using individual player box scores)
- [ ] Implement parlay bet grading (all legs must win)
- [ ] Implement bankroll management rules: max bet size, Kelly criterion sizing
- [ ] Implement calibration curve generation

## Dependencies

- **statistics-service** (Phase 1) for final game results (bet grading)
- **lines-service** (Phase 1) for closing lines (CLV computation)
- **Redis** for event subscription and pub/sub
- **Postgres** for bet and performance storage

## Complexity

**M** -- Well-defined domain logic (bet placement, grading, analytics). Grading logic has sport-specific rules that add complexity, but the core is straightforward CRUD + computation.

## Definition of Done

- [ ] `POST /api/v1/bets` successfully places a paper bet and persists it
- [ ] Bets are graded correctly when games complete (WON/LOST/PUSH)
- [ ] Push scenarios return the stake correctly
- [ ] `GET /api/v1/performance` returns ROI, win rate, CLV, and units
- [ ] `GET /api/v1/bets` supports filtering by sport, bet type, date range, and outcome
- [ ] CLV is computed by comparing placement odds to closing odds
- [ ] Event-driven grading works via Redis `game.completed` subscription
- [ ] Polling fallback catches any missed events
- [ ] Alembic migrations run successfully
- [ ] Tests pass

## Key Documentation

- [Bookie Emulator Component](../bookie-breaker-docs/components/bookie-emulator.md)
- [Feature Inventory: PAPER-001 through PAPER-032](../bookie-breaker-docs/architecture/feature-inventory.md)
- [Database Schemas](../bookie-breaker-docs/schemas/database-schemas/)
