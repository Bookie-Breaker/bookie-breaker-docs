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

| Source                    | Data                                | Mechanism                   |
| ------------------------- | ----------------------------------- | --------------------------- |
| MCP client (e.g., Claude) | Tool calls with parameters          | MCP protocol (stdio or SSE) |
| agent                     | Analysis, edges, pipeline status    | API response                |
| bookie-emulator           | Paper bet data, performance metrics | API response                |
| lines-service             | Lines and odds data                 | API response                |
| simulation-engine         | Simulation results                  | API response                |
| prediction-engine         | Prediction results                  | API response                |
| statistics-service        | Stats data                          | API response                |

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
- **FR-012:** Support both stdio and SSE transport modes for MCP communication.
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

The MCP server does not expose REST APIs. It exposes **MCP tools** and **MCP resources** via the MCP protocol (stdio or
SSE on port 8007):

| MCP Tool              | Description                       | Delegates To                                               |
| --------------------- | --------------------------------- | ---------------------------------------------------------- |
| `get_edges`           | List detected edges with filters  | agent `GET /api/v1/edges`                                  |
| `get_edge_detail`     | Get detailed edge with analysis   | agent `GET /api/v1/edges/{edge_id}`                        |
| `get_predictions`     | Get predictions for a game        | prediction-engine `GET /api/v1/predictions/game/{game_id}` |
| `get_lines`           | Get current lines for a game      | lines-service `GET /api/v1/lines/{game_id}`                |
| `get_line_movement`   | Get line movement for a game      | lines-service `GET /api/v1/lines/{game_id}/movement`       |
| `get_team_stats`      | Get stats for a team              | statistics-service `GET /api/v1/teams/{team_id}/stats`     |
| `get_player_stats`    | Get stats for a player            | statistics-service `GET /api/v1/players/{player_id}/stats` |
| `get_simulation`      | Get simulation results for a game | simulation-engine `GET /api/v1/simulations/game/{game_id}` |
| `place_bet`           | Place a paper bet                 | bookie-emulator `POST /api/v1/bets`                        |
| `get_bets`            | List paper bets with filters      | bookie-emulator `GET /api/v1/bets`                         |
| `get_performance`     | Get performance metrics           | bookie-emulator `GET /api/v1/performance`                  |
| `ask_analyst`         | Ask an analytical question        | agent `POST /api/v1/query`                                 |
| `run_pipeline`        | Trigger a pipeline run            | agent `POST /api/v1/pipeline/run`                          |
| `get_pipeline_status` | Check pipeline status             | agent `GET /api/v1/pipeline/status`                        |
| `get_health`          | Check system health               | agent `GET /api/v1/health`                                 |

| MCP Resource                          | Description                      | Source                                            |
| ------------------------------------- | -------------------------------- | ------------------------------------------------- |
| `bookiebreaker://edges/current`       | Summary of current active edges  | agent `GET /api/v1/edges?is_stale=false`          |
| `bookiebreaker://games/today`         | Today's games across all leagues | statistics-service `GET /api/v1/games?date=today` |
| `bookiebreaker://performance/summary` | Recent performance snapshot      | bookie-emulator `GET /api/v1/performance`         |

### APIs Consumed

| Service            | Endpoint                                 | Purpose                                                   |
| ------------------ | ---------------------------------------- | --------------------------------------------------------- |
| agent              | `GET /api/v1/edges`                      | Fetch edges for `get_edges` tool                          |
| agent              | `GET /api/v1/edges/{edge_id}`            | Fetch edge detail for `get_edge_detail` tool              |
| agent              | `POST /api/v1/query`                     | Submit questions for `ask_analyst` tool                   |
| agent              | `POST /api/v1/pipeline/run`              | Trigger pipeline for `run_pipeline` tool                  |
| agent              | `GET /api/v1/pipeline/status`            | Pipeline status for `get_pipeline_status` tool            |
| agent              | `GET /api/v1/health`                     | System health for `get_health` tool                       |
| bookie-emulator    | `POST /api/v1/bets`                      | Place bets for `place_bet` tool                           |
| bookie-emulator    | `GET /api/v1/bets`                       | Fetch bets for `get_bets` tool                            |
| bookie-emulator    | `GET /api/v1/performance`                | Fetch metrics for `get_performance` tool                  |
| lines-service      | `GET /api/v1/lines/{game_id}`            | Fetch lines for `get_lines` tool                          |
| lines-service      | `GET /api/v1/lines/{game_id}/movement`   | Fetch movement for `get_line_movement` tool               |
| statistics-service | `GET /api/v1/teams/{team_id}/stats`      | Fetch stats for `get_team_stats` tool                     |
| statistics-service | `GET /api/v1/players/{player_id}/stats`  | Fetch stats for `get_player_stats` tool                   |
| statistics-service | `GET /api/v1/games`                      | Fetch schedule for `bookiebreaker://games/today` resource |
| simulation-engine  | `GET /api/v1/simulations/game/{game_id}` | Fetch simulation for `get_simulation` tool                |
| prediction-engine  | `GET /api/v1/predictions/game/{game_id}` | Fetch predictions for `get_predictions` tool              |

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
