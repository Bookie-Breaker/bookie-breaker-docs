# bookie-breaker-mcp-server

## Purpose

The single centralized MCP gateway for the entire BookieBreaker system. Exposes all system capabilities as MCP tools,
enabling natural language interaction through any MCP-compatible client (e.g., Claude Desktop, Claude Code). One of
three equal first-class interfaces alongside CLI and UI. No other service exposes MCP tools directly -- all MCP access
is routed through this server to control context size and tool grouping for LLM interactions.

## Responsibilities

- Exposes all system operations as MCP tools: viewing edges, querying predictions, placing paper bets, checking
  performance, looking up lines and stats, and asking the LLM analyst questions.
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

| Source                    | Data                                | Mechanism                               |
| ------------------------- | ----------------------------------- | --------------------------------------- |
| MCP client (e.g., Claude) | Tool calls with parameters          | MCP protocol (stdio or streamable HTTP) |
| agent                     | Analysis, edges, pipeline status    | API response                            |
| bookie-emulator           | Paper bet data, performance metrics | API response                            |
| lines-service             | Lines and odds data                 | API response                            |
| simulation-engine         | Simulation results                  | API response                            |
| prediction-engine         | Prediction results                  | API response                            |
| statistics-service        | Stats data                          | API response                            |

## Outputs

| Destination        | Data                                             | Mechanism             |
| ------------------ | ------------------------------------------------ | --------------------- |
| MCP client         | Tool results (structured text, tables, analysis) | MCP protocol response |
| agent              | User questions, pipeline commands                | API call              |
| bookie-emulator    | Paper bet placement requests                     | API call              |
| lines-service      | Line lookup requests                             | API call              |
| statistics-service | Stats lookup requests                            | API call              |

## Dependencies

- **agent** -- for LLM analysis, edge detection, pipeline orchestration
- **bookie-emulator** -- for paper trading operations and performance data
- **lines-service** -- for current and historical lines
- **simulation-engine** -- for simulation results
- **prediction-engine** -- for prediction results
- **statistics-service** -- for stats lookups

## Dependents

None. The MCP server is a leaf node -- it is consumed by external MCP clients and has no downstream service dependents
within BookieBreaker.

---

## Requirements

### Functional Requirements

- **FR-001:** Expose MCP tools for viewing detected edges with filtering by league, date, minimum edge size, market
  type, and staleness (e.g., `get_edges(league="NBA", min_edge=3.0)`).
- **FR-002:** Expose MCP tools for querying predictions for specific games with calibrated probabilities, confidence
  intervals, and feature importance.
- **FR-003:** Expose MCP tools for placing paper bets through the bookie-emulator (e.g., `place_bet(game_id,
market_type, selection, stake)`).
- **FR-004:** Expose MCP tools for querying paper trading performance metrics with breakdowns by league, bet type, and
  time window.
- **FR-005:** Expose MCP tools for looking up current betting lines and line movement for any game.
- **FR-006:** Expose MCP tools for looking up team and player statistics.
- **FR-007:** Expose MCP tools for submitting analytical questions to the agent's LLM analyst (e.g.,
  `ask_analyst(question="Why do you like the over in Lakers-Celtics?")`).
- **FR-008:** Expose MCP tools for triggering and checking pipeline status.
- **FR-009:** Expose MCP tools for checking system health across all services.
- **FR-010:** Provide MCP resources for contextual data that LLM clients can read without a tool call: current edges
  summary, today's games, recent performance snapshot.
- **FR-011:** Implement proper MCP protocol lifecycle: initialization, capability negotiation, tool listing with JSON
  schemas, and parameter validation.
- **FR-012:** Support both stdio and streamable HTTP transport modes for MCP communication (SSE deprecated
  by the MCP spec; see [ADR-022](../decisions/022-mcp-sdk-and-transport.md)).
- **FR-013:** Format all tool responses as structured but LLM-readable text (markdown tables, clear labels, concise
  summaries) optimized for LLM consumption rather than raw JSON.
- **FR-014:** Validate all incoming tool call parameters against defined JSON schemas and return clear error messages
  for invalid inputs.

### Non-Functional Requirements

- **Latency:** Tool call response for data lookups (edges, lines, stats, bets): < 2 seconds. Tool call response for
  analytical questions (LLM): < 10 seconds. Tool listing / capability negotiation: < 500ms.
- **Throughput:** Handle up to 10 concurrent MCP tool calls from a single MCP client. Typical usage: 1-5 tool calls per
  minute during active use.
- **Availability:** Available whenever the underlying backend services and the mcp-server container are running. No
  independent high-availability requirement. Returns clear error messages when backend services are unreachable.
- **Storage:** None. The MCP server is completely stateless.

### Data Ownership

None. The MCP server does not own any domain entities. It is a stateless protocol adapter that translates MCP tool calls
into backend API calls.

### APIs Exposed

The MCP server does not expose REST APIs. It exposes **MCP tools** and **MCP resources** via the MCP protocol
(stdio, or streamable HTTP at `/mcp` on port 8007 per
[ADR-022](../decisions/022-mcp-sdk-and-transport.md); a plain `GET /health` route serves the container
healthcheck):

| MCP Tool              | Description                            | Delegates To                                               |
| --------------------- | -------------------------------------- | ---------------------------------------------------------- |
| `get_edges`           | List detected edges with filters       | agent `GET /api/v1/agent/edges`                            |
| `get_edge_detail`     | Get detailed edge with analysis        | agent `GET /api/v1/agent/edges/{edge_id}`                  |
| `get_slate`           | Today's games with predictions/edges   | agent `GET /api/v1/agent/slate`                            |
| `get_prediction`      | Latest predictions for a game          | prediction-engine `GET /api/v1/predict/games/{id}/latest`  |
| `get_lines`           | Current lines, or movement history     | lines-service `GET /api/v1/lines/current` / `.../movement` |
| `get_team_stats`      | Team statistics                        | statistics-service `GET /api/v1/stats/teams`               |
| `get_player_stats`    | Player statistics                      | statistics-service `GET /api/v1/stats/players`             |
| `get_simulation`      | Latest simulation results for a game   | simulation-engine `GET /api/v1/sim/games/{id}/latest`      |
| `place_bet`           | Paper-bet a detected edge (idempotent) | bookie-emulator `POST /api/v1/emulator/bets`               |
| `get_bet_history`     | Paper bet ledger with filters          | bookie-emulator `GET /api/v1/emulator/bets`                |
| `get_performance`     | Performance metrics                    | bookie-emulator `GET /api/v1/emulator/performance`         |
| `ask_analyst`         | Ask the LLM analyst (scoped)           | agent `POST /api/v1/agent/analysis`                        |
| `run_pipeline`        | Trigger a pipeline run                 | agent `POST /api/v1/agent/pipeline/run`                    |
| `get_pipeline_status` | Check a pipeline run                   | agent `GET /api/v1/agent/pipeline/runs/{id}`               |
| `get_health`          | Backend health fan-out                 | every backend's health endpoint                            |

| MCP Resource                          | Description                     | Source                                             |
| ------------------------------------- | ------------------------------- | -------------------------------------------------- |
| `bookiebreaker://edges/current`       | Summary of current active edges | agent `GET /api/v1/agent/edges` (fresh only)       |
| `bookiebreaker://games/today`         | Today's slate                   | agent `GET /api/v1/agent/slate`                    |
| `bookiebreaker://performance/summary` | Recent performance snapshot     | bookie-emulator `GET /api/v1/emulator/performance` |

`place_bet` takes an `edge_id` + stake, builds the bet body from the agent's edge detail (rejecting stale
edges), and sends a deterministic `X-Idempotency-Key` (UUIDv5 over edge + stake) so retries never double-bet.
`ask_analyst` infers the analysis type from its scope: `edge_id` → `EDGE_BREAKDOWN`, `game_id` →
`GAME_PREVIEW`, unscoped → `PERFORMANCE_REVIEW`.

### APIs Consumed

| Service            | Endpoint                                     | Purpose                                                |
| ------------------ | -------------------------------------------- | ------------------------------------------------------ |
| agent              | `GET /api/v1/agent/edges`                    | `get_edges` tool, `edges/current` resource             |
| agent              | `GET /api/v1/agent/edges/{edge_id}`          | `get_edge_detail` tool, `place_bet` edge lookup        |
| agent              | `GET /api/v1/agent/slate`                    | `get_slate` tool, `games/today` resource               |
| agent              | `POST /api/v1/agent/analysis`                | `ask_analyst` tool (120s timeout for LLM generation)   |
| agent              | `POST /api/v1/agent/pipeline/run`            | `run_pipeline` tool                                    |
| agent              | `GET /api/v1/agent/pipeline/runs/{id}`       | `get_pipeline_status` tool                             |
| bookie-emulator    | `POST /api/v1/emulator/bets`                 | `place_bet` tool                                       |
| bookie-emulator    | `GET /api/v1/emulator/bets`                  | `get_bet_history` tool                                 |
| bookie-emulator    | `GET /api/v1/emulator/performance`           | `get_performance` tool, `performance/summary` resource |
| lines-service      | `GET /api/v1/lines/current`                  | `get_lines` tool                                       |
| lines-service      | `GET /api/v1/lines/game/{id}/movement`       | `get_lines --movement`                                 |
| statistics-service | `GET /api/v1/stats/teams`                    | `get_team_stats` tool                                  |
| statistics-service | `GET /api/v1/stats/players`                  | `get_player_stats` tool                                |
| simulation-engine  | `GET /api/v1/sim/games/{game_id}/latest`     | `get_simulation` tool                                  |
| prediction-engine  | `GET /api/v1/predict/games/{game_id}/latest` | `get_prediction` tool                                  |
| all services       | health endpoints                             | `get_health` tool                                      |

### Events Published

None. The MCP server does not publish events.

### Events Subscribed

None. The MCP server is request-driven (responds to MCP tool calls). It does not subscribe to Redis events. Real-time
updates are handled by the MCP client polling or by the user invoking tools.

### Storage Requirements

- **Database:** None. The MCP server is completely stateless.
- **No persistent storage.** All data is fetched from backend services on each tool call.
- **In-memory:** Maintains MCP session state (capabilities, tool schemas) for the duration of a client connection.
  Negligible memory footprint.
