# PLANNING: mcp-server

## Service
- **Name:** mcp-server
- **Language:** Python 3.12+
- **Framework:** MCP SDK (FastMCP)

## Implementation Phase
Phase 4 (Agent Intelligence & MCP)

## Purpose
Exposes BookieBreaker capabilities as MCP (Model Context Protocol) tools, allowing IDE assistants (Claude Desktop, VS Code) and other MCP-compatible clients to query predictions, edges, stats, and place paper bets programmatically.

## Ordered Task List

1. Initialize Python project: `pyproject.toml` with uv, `src/` layout
2. Set up FastMCP server with protocol lifecycle (initialization, capability negotiation)
3. Implement HTTP clients for backend services (agent, lines-service, statistics-service, bookie-emulator)
4. Implement MCP tools:
   - `get_edges` -- current edges with filters (sport, bet type, min edge)
   - `get_prediction` -- prediction details for a specific game
   - `get_slate` -- today's games with predictions
   - `get_lines` -- current lines for a game
   - `get_stats` -- team or player statistics
   - `place_bet` -- place a paper bet
   - `get_performance` -- paper trading performance summary
   - `get_bet_history` -- paper bet history with filters
   - `ask_analyst` -- send a question to the LLM analyst
   - `run_pipeline` -- trigger a pipeline run
   - `get_pipeline_status` -- check pipeline status
   - `get_simulation` -- simulation results for a matchup
5. Implement MCP resources:
   - `edges://current` -- current edges summary (contextual data for LLM)
   - `performance://recent` -- recent paper trading performance
6. Format all responses for LLM consumption: structured but readable, no raw JSON dumps
7. Support both stdio and SSE transport modes
8. Write integration tests: verify each tool returns expected response shape
9. Test with Claude Desktop and/or VS Code MCP extension
10. Create Dockerfile and integrate into Docker Compose
11. Add `.env.example`

## Dependencies
- **agent** (Phase 3/4) for edges, analysis, pipeline operations
- **lines-service** (Phase 1) for direct line lookups
- **statistics-service** (Phase 1) for direct stat lookups
- **bookie-emulator** (Phase 3) for bet placement and performance

## Complexity
**M** -- Primarily a thin translation layer between MCP protocol and existing REST APIs. The MCP SDK handles protocol complexity. Main work is in formatting responses for LLM consumption and testing with real MCP clients.

## Definition of Done
- [ ] MCP server responds to `tools/list` with all implemented tools
- [ ] `get_edges` tool returns current edges when called from an MCP client
- [ ] `place_bet` tool successfully places a paper bet
- [ ] `ask_analyst` tool returns LLM-generated analysis
- [ ] Both stdio and SSE transports work
- [ ] Resources provide contextual data
- [ ] Responses are formatted for LLM readability
- [ ] Tested with at least one real MCP client (Claude Desktop, VS Code, or equivalent)

## Key Documentation
- [MCP Server Component](../bookie-breaker-docs/components/mcp-server.md)
- [Feature Inventory: MCP-001 through MCP-018](../bookie-breaker-docs/architecture/feature-inventory.md)
- [Communication Patterns](../bookie-breaker-docs/architecture/communication-patterns.md)
