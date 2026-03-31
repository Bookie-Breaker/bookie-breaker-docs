# bookie-breaker-mcp-server

## Purpose

Model Context Protocol (MCP) server that exposes the BookieBreaker system's full capabilities as MCP tools, enabling natural language interaction with the system through any MCP-compatible client (e.g., Claude Desktop, Claude Code). One of three equal first-class interfaces alongside CLI and UI.

## Responsibilities

- Exposes all system operations as MCP tools: viewing edges, querying predictions, placing paper bets, checking performance, looking up lines and stats, and asking the LLM analyst questions.
- Translates MCP tool calls into the appropriate backend API calls.
- Formats responses in a way that is useful for LLM consumption (structured but readable).
- Manages MCP tool schemas, descriptions, and parameter validation.
- Handles MCP protocol lifecycle (initialization, capability negotiation, tool listing).
- Provides MCP resources for contextual data (e.g., current edges, recent performance summary).

## Non-Responsibilities

- Does NOT contain business logic. All computation happens in backend services.
- Does NOT store data persistently. It is a stateless protocol adapter.
- Does NOT own the LLM that interprets responses. The MCP client (e.g., Claude) handles that.
- Does NOT duplicate the agent's LLM capabilities. For analysis questions, it delegates to the agent.
- Does NOT define API contracts for backend services. It consumes their existing APIs.

## Inputs

| Source | Data | Mechanism |
|---|---|---|
| MCP client (e.g., Claude) | Tool calls with parameters | MCP protocol (stdio or SSE) |
| agent | Analysis, edges, pipeline status | API response |
| bookie-emulator | Paper bet data, performance metrics | API response |
| lines-service | Lines and odds data | API response |
| simulation-engine | Simulation results | API response |
| prediction-engine | Prediction results | API response |
| statistics-service | Stats data | API response |

## Outputs

| Destination | Data | Mechanism |
|---|---|---|
| MCP client | Tool results (structured text, tables, analysis) | MCP protocol response |
| agent | User questions, pipeline commands | API call |
| bookie-emulator | Paper bet placement requests | API call |
| lines-service | Line lookup requests | API call |
| statistics-service | Stats lookup requests | API call |

## Dependencies

- **agent** -- for LLM analysis, edge detection, pipeline orchestration
- **bookie-emulator** -- for paper trading operations and performance data
- **lines-service** -- for current and historical lines
- **simulation-engine** -- for simulation results
- **prediction-engine** -- for prediction results
- **statistics-service** -- for stats lookups

## Dependents

None. The MCP server is a leaf node -- it is consumed by external MCP clients and has no downstream service dependents within BookieBreaker.
