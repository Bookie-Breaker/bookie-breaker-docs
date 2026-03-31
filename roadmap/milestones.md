# Milestones

Concise tracker for BookieBreaker implementation milestones. Each milestone marks the completion of a phase and gates the start of dependent work.

---

## M1: NBA Data Pipeline

| Field | Value |
|-------|-------|
| **Phase** | 1 -- Infrastructure & Data Foundation |
| **Key Deliverables** | Docker Compose stack running Postgres+TimescaleDB and Redis; statistics-service serving NBA stats with Redis caching; lines-service ingesting and serving NBA lines from The Odds API; OpenAPI specs for both services; seed data script |
| **Success Criteria** | `task up` starts the full infrastructure stack. `GET /api/v1/stats/nba/teams` returns live NBA team stats. `GET /api/v1/lines/current?sport=basketball_nba` returns current NBA odds. Line snapshots accumulate in TimescaleDB over time. Integration tests pass. |
| **Dependencies** | None |

---

## M2: NBA Predictions & Edge Detection

| Field | Value |
|-------|-------|
| **Phase** | 2 -- Prediction Core |
| **Key Deliverables** | simulation-engine with basketball Monte Carlo plugin; prediction-engine with trained NBA XGBoost model and calibrated probability output; edge detection math module; OpenAPI codegen pipeline |
| **Success Criteria** | Simulation produces NBA score distributions (10K+ iterations) with convergence. Prediction-engine returns calibrated probabilities with confidence intervals for spread, total, and moneyline. Edge detection identifies games where calibrated probability exceeds market-implied probability. |
| **Dependencies** | M1 (statistics-service and lines-service operational) |

---

## M3: NBA End-to-End Workflow

| Field | Value |
|-------|-------|
| **Phase** | 3 -- First Interface & Paper Trading |
| **Key Deliverables** | bookie-emulator with bet placement, grading, and performance analytics; agent with pipeline orchestration and edge detection; CLI with core commands (`edges`, `bet`, `performance`, `lines`, `slate`) |
| **Success Criteria** | User can run `bb edges` to see NBA edges, `bb bet place` to paper bet, `bb performance` to see ROI/win rate/CLV. Agent runs scheduled pipeline. Bookie-emulator grades bets on game completion. Full end-to-end test passes: data ingestion through bet grading. |
| **Dependencies** | M2 (simulation and prediction engines operational) |

---

## M4: LLM Analysis & MCP Integration

| Field | Value |
|-------|-------|
| **Phase** | 4 -- Agent Intelligence & MCP |
| **Key Deliverables** | Agent with Anthropic SDK integration, natural language edge analysis, daily summaries; MCP server with full tool set (edges, predictions, bets, analysis); CLI `ask` command |
| **Success Criteria** | `bb ask "Why do you like the over?"` returns coherent LLM analysis. Agent generates automated daily summaries. MCP server responds to tool calls from Claude Desktop or equivalent client. Both stdio and SSE transports work. |
| **Dependencies** | M3 (agent orchestration and all backend services running) |

---

## M5: Web Dashboard

| Field | Value |
|-------|-------|
| **Phase** | 5 -- Dashboard |
| **Key Deliverables** | SvelteKit dashboard with edges table, prediction details, line movement charts, simulation distributions, paper trading performance charts, bet ledger, bet placement form, LLM chat interface, live updates via Redis |
| **Success Criteria** | Dashboard loads in browser at `localhost:3000`. Edges table shows current NBA edges with filters. Line movement chart renders for any game. Performance dashboard shows ROI over time. User can place a paper bet through the UI. Live updates appear without page refresh. Playwright tests pass. |
| **Dependencies** | M4 (agent LLM analysis for chat interface) |

---

## M6: All Leagues Operational

| Field | Value |
|-------|-------|
| **Phase** | 6 -- Sport Expansion |
| **Key Deliverables** | Statistics adapters for NFL, MLB, NCAA Basketball, NCAA Football, NCAA Baseball; football and baseball simulation plugins; sport-specific ML models; multi-sport edge detection; multi-sport paper trading; league selectors in CLI and UI |
| **Success Criteria** | All 6 leagues return stats from statistics-service. Lines are ingested for all 6 leagues. Simulation-engine runs football, basketball, and baseball simulations. Edges are detected across all leagues. Paper bets can be placed and graded for all leagues. CLI `bb edges --sport nfl` and UI league filter both work. |
| **Dependencies** | M3 (NBA end-to-end pipeline proven out) |

---

## M7: Full Bet Type Coverage

| Field | Value |
|-------|-------|
| **Phase** | 7 -- Advanced Features |
| **Key Deliverables** | Player prop modeling and edge detection; live/in-game betting via SharpAPI SSE; parlay correlation analysis; ensemble ML models; model A/B testing framework |
| **Success Criteria** | Player prop edges detected for NBA and NFL. Live lines update via SSE during games. Live simulation updates produce in-game edges. Parlay correlation identifies mispriced same-game parlays. A/B testing framework compares model variants with tracked performance. All new bet types can be paper traded. |
| **Dependencies** | M6 (all leagues operational) |

---

## Milestone Dependency Graph

```
M1 ──> M2 ──> M3 ──> M4 ──> M5
                │
                └───────────> M6 ──> M7
```

Phases 4 and 5 (M4, M5) can proceed in parallel with Phase 6 (M6) after M3 is complete. Phase 7 (M7) requires M6.

---

## Progress Tracker

| Milestone | Status | Started | Completed |
|-----------|--------|---------|-----------|
| M1: NBA Data Pipeline | Not Started | -- | -- |
| M2: NBA Predictions & Edge Detection | Not Started | -- | -- |
| M3: NBA End-to-End Workflow | Not Started | -- | -- |
| M4: LLM Analysis & MCP Integration | Not Started | -- | -- |
| M5: Web Dashboard | Not Started | -- | -- |
| M6: All Leagues Operational | Not Started | -- | -- |
| M7: Full Bet Type Coverage | Not Started | -- | -- |
