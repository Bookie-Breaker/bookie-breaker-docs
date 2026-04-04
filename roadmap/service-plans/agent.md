# PLANNING: agent

## Service
- **Name:** agent
- **Language:** Python 3.12+
- **Framework:** FastAPI

## Implementation Phase
Phase 3 (First Interface & Paper Trading) -- initial orchestration
Phase 4 (Agent Intelligence & MCP) -- LLM integration and enhanced intelligence

## Purpose
Central coordinator of the BookieBreaker system. Orchestrates the prediction pipeline (data ingestion through edge detection), performs edge detection by comparing calibrated probabilities against market odds, and serves as the query gateway for all three user interfaces. In Phase 4, gains LLM-powered analysis capabilities via the Anthropic SDK.

## Ordered Task List

**Phase 3 (pipeline orchestration):**
- [ ] Initialize Python project: `pyproject.toml` with uv, `src/` layout, FastAPI app scaffold
- [ ] Set up FastAPI server with uvicorn, CORS middleware, request logging
- [ ] Implement health check endpoint (`GET /healthz`)
- [ ] Implement HTTP clients for all backend services (httpx async): statistics-service, simulation-engine, prediction-engine, lines-service, bookie-emulator
- [ ] Implement edge detection logic:
  - [ ] Fetch calibrated probabilities from prediction-engine
  - [ ] Fetch current lines from lines-service
  - [ ] Convert odds to implied probabilities
  - [ ] Compare: edge = calibrated_probability - implied_probability
  - [ ] Filter by configurable minimum edge threshold
  - [ ] Rank edges by expected value
- [ ] Implement pipeline orchestration:
  - [ ] Sequence: fetch stats → run simulations → generate predictions → detect edges
  - [ ] Handle failures gracefully with per-step error reporting
  - [ ] Track pipeline run status (running, completed, failed)
- [ ] Implement scheduled pipeline runs (APScheduler v4, per [ADR-015](../../decisions/015-pipeline-scheduler.md))
- [ ] Implement auto-bet: when edge exceeds threshold, place paper bet via bookie-emulator
- [ ] Implement Redis pub/sub: subscribe to `lines.updated`, `stats.updated`, `game.completed`; publish `edge.detected`, `prediction.completed`
- [ ] Build REST API:
  - [ ] `GET /api/v1/edges` -- current edges with filters (sport, bet type, min edge)
  - [ ] `POST /api/v1/pipeline/run` -- trigger on-demand pipeline run
  - [ ] `GET /api/v1/pipeline/status` -- current pipeline status
  - [ ] `GET /api/v1/slate` -- today's games with predictions and edges
- [ ] Write integration tests for pipeline orchestration
- [ ] Create Dockerfile and integrate into Docker Compose
- [ ] Add `.env.example`

**Phase 4 (LLM intelligence):**
- [ ] Integrate Anthropic SDK (anthropic Python package)
- [ ] Design prompt templates:
  - [ ] Edge analysis: "Why does this edge exist? What factors drive it?"
  - [ ] Game preview: key matchup factors and betting angles
  - [ ] Performance commentary: interpret paper trading trends
  - [ ] General Q&A: answer user questions about bets and predictions
- [ ] Implement `POST /api/v1/analyze` -- accepts question + optional context (game ID, edge ID), returns LLM-generated analysis
- [ ] Implement daily summary generation: automated edge digest with commentary
- [ ] Implement configurable pipeline schedules (e.g., "2 hours before first game")
- [ ] Add retry logic and circuit breakers for service calls
- [ ] Enhance alerting: push notifications via Redis pub/sub with natural language descriptions

## Dependencies
- **statistics-service** (Phase 1) for stats data
- **lines-service** (Phase 1) for odds data
- **simulation-engine** (Phase 2) for simulation results
- **prediction-engine** (Phase 2) for calibrated probabilities
- **bookie-emulator** (Phase 3, built concurrently) for paper bet placement
- **Anthropic API key** (Phase 4) for LLM calls

## Complexity
**XL** -- Coordinates all other services, implements edge detection business logic, manages pipeline scheduling, and integrates LLM capabilities. Failure handling across multiple services is the primary complexity driver.

## Definition of Done

**Phase 3:**
- [ ] `GET /api/v1/edges` returns detected NBA edges with edge size and ranking
- [ ] `POST /api/v1/pipeline/run` triggers full pipeline and returns results
- [ ] Pipeline sequences all service calls correctly
- [ ] Auto-bet places paper bets on edges above threshold
- [ ] Redis events are published and consumed
- [ ] Pipeline failures are reported clearly

**Phase 4:**
- [ ] `POST /api/v1/analyze` returns coherent LLM-generated analysis
- [ ] Daily summary is generated automatically
- [ ] Scheduled pipeline runs execute at configured times
- [ ] Retry logic handles transient service failures

## Key Documentation
- [Agent Component](../bookie-breaker-docs/components/agent.md)
- [Edge Detection](../bookie-breaker-docs/algorithms/edge-detection.md)
- [Data Flow Architecture](../bookie-breaker-docs/architecture/data-flow.md)
- [Communication Patterns](../bookie-breaker-docs/architecture/communication-patterns.md)
- [Feature Inventory: PIPE-040 through PIPE-054, AGT-001 through AGT-018](../bookie-breaker-docs/architecture/feature-inventory.md)
