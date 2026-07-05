# PLANNING: mcp-server

## Service

- **Name:** mcp-server
- **Language:** Python 3.12+
- **Framework:** FastMCP (standalone `fastmcp` package over the official `mcp` SDK, per
  [ADR-022](../../decisions/022-mcp-sdk-and-transport.md))

## Implementation Phase

Phase 4 (Agent Intelligence & MCP) — **implemented 2026-07-05** except the real-MCP-client test
(verification session).

## Purpose

Exposes BookieBreaker capabilities as MCP (Model Context Protocol) tools, allowing IDE assistants (Claude
Desktop, Claude Code, VS Code) and other MCP-compatible clients to query predictions, edges, stats, and place
paper bets programmatically. A stateless REST-to-MCP bridge: no database, no business logic.

## Ordered Task List

- [x] Initialize Python project: `pyproject.toml` with uv, `src/` layout
- [x] Set up FastMCP server with protocol lifecycle (initialization, capability negotiation)
- [x] Implement HTTP clients for backend services (agent, lines-service, statistics-service,
      simulation-engine, prediction-engine, bookie-emulator)
- [x] Implement MCP tools (canonical list — 15):
  - [x] `get_edges` -- current edges with filters (league, market type, min edge, date, stale)
  - [x] `get_edge_detail` -- one edge with game, prediction, line, paper bet, stored analysis
  - [x] `get_slate` -- today's games with predictions and edge counts
  - [x] `get_prediction` -- latest calibrated predictions for a game
  - [x] `get_lines` -- current lines across books, or one game's movement history
  - [x] `get_team_stats` / `get_player_stats` -- statistics-service lookups
  - [x] `get_simulation` -- latest simulation results for a game
  - [x] `place_bet` -- place a paper bet on a detected edge (idempotent per edge + stake)
  - [x] `get_performance` -- paper trading performance summary
  - [x] `get_bet_history` -- paper bet ledger with filters
  - [x] `ask_analyst` -- question to the LLM analyst via `POST /api/v1/agent/analysis`
        (type inferred from edge/game scope)
  - [x] `run_pipeline` / `get_pipeline_status` -- trigger and poll pipeline runs
  - [x] `get_health` -- backend health fan-out
- [x] Implement MCP resources (`bookiebreaker://` scheme):
  - [x] `bookiebreaker://edges/current` -- current edges summary (contextual data for LLM)
  - [x] `bookiebreaker://performance/summary` -- recent paper trading performance
  - [x] `bookiebreaker://games/today` -- today's slate
- [x] Format all responses for LLM consumption: markdown tables and sections, no raw JSON dumps
- [x] Support stdio and **streamable HTTP** transports (SSE deprecated by the MCP spec; ADR-022)
- [x] Write integration tests: real stdio subprocess + streamable HTTP against a stub backend
      (initialize → tools/list → tool calls → resource reads)
- [ ] Test with Claude Desktop and/or Claude Code — **verification session**
- [x] Create Dockerfile and integrate into Docker Compose (streamable HTTP on port 8007)
- [x] Add `.env.example`

## Dependencies

- **agent** (Phase 3/4) for edges, slate, analysis, pipeline operations
- **lines-service** (Phase 1) for direct line lookups
- **statistics-service** (Phase 1) for direct stat lookups
- **simulation-engine** / **prediction-engine** (Phase 2) for simulation and prediction lookups
- **bookie-emulator** (Phase 3) for bet placement and performance

## Complexity

**M** -- Primarily a thin translation layer between MCP protocol and existing REST APIs. The MCP SDK handles
protocol complexity. Main work is in formatting responses for LLM consumption and testing with real MCP
clients.

## Definition of Done

- [x] MCP server responds to `tools/list` with all implemented tools (15; asserted over both transports)
- [x] `get_edges` tool returns current edges when called from an MCP client (in-memory + stdio + HTTP clients)
- [x] `place_bet` tool successfully places a paper bet
- [x] `ask_analyst` tool returns LLM-generated analysis (respx-mocked agent; live LLM in verification session)
- [x] stdio and streamable HTTP transports work (SSE superseded per ADR-022)
- [x] Resources provide contextual data
- [x] Responses are formatted for LLM readability
- [ ] Tested with at least one real MCP client (Claude Desktop, Claude Code, or equivalent) —
      **verification session**

## Key Documentation

- [MCP Server Component](../../components/mcp-server.md)
- [ADR-022: MCP SDK and Transport](../../decisions/022-mcp-sdk-and-transport.md)
- [Feature Inventory: MCP-001 through MCP-018](../../architecture/feature-inventory.md)
- [Communication Patterns](../../architecture/communication-patterns.md)
