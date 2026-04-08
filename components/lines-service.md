# bookie-breaker-lines-service

## Purpose

Ingests, normalizes, stores, and serves betting lines and odds from external APIs. Serves as the system's source of truth for current and historical lines across all supported bet types and leagues, and tracks line movement over time.

## Responsibilities

- Ingests lines/odds from external odds APIs (e.g., The Odds API, or similar) on a regular polling schedule.
- Normalizes lines into a canonical format across different source APIs and sportsbooks.
- Stores historical lines with timestamps to enable line movement tracking.
- Serves current lines/odds for any supported game, bet type, and sportsbook.
- Serves historical line movement data (how a line has moved from open to current).
- Covers all bet types: spreads, totals (over/under), moneylines, player props, team props, futures.
- Covers all 6 leagues: NFL, NBA, MLB, NCAA Football, NCAA Basketball, NCAA Baseball.
- Provides closing lines (final line before game start) for CLV calculations.
- Identifies sharp line moves and steam moves when detectable.

## Non-Responsibilities

- Does NOT calculate implied probabilities or detect edges. The prediction-engine and agent own that.
- Does NOT store or serve game results or statistics. That belongs to statistics-service.
- Does NOT decide which lines represent value. It serves the raw market data.
- Does NOT place real or paper bets. The bookie-emulator handles paper trading.
- Does NOT perform any predictive modeling or simulation.

## Inputs

| Source                | Data                                                                   | Mechanism             |
| --------------------- | ---------------------------------------------------------------------- | --------------------- |
| External odds APIs    | Raw lines/odds across sportsbooks                                      | Scheduled API polling |
| agent                 | Requests for specific lines, line movement, closing lines              | API call              |
| bookie-emulator       | Requests for current odds at bet placement time, closing lines for CLV | API call              |
| CLI / UI / MCP server | Line lookup and line movement requests                                 | API call              |

## Outputs

| Destination       | Data                                                                | Mechanism    |
| ----------------- | ------------------------------------------------------------------- | ------------ |
| agent             | Current lines, historical lines, line movement data                 | API response |
| prediction-engine | Market lines for edge calculation (predicted prob vs. implied prob) | API response |
| bookie-emulator   | Current odds at placement time, closing lines                       | API response |
| CLI / UI          | Lines, odds, line movement charts data                              | API response |

## Dependencies

- **External odds APIs** (external) -- source of all raw lines/odds data

## Dependents

- **bookie-breaker-agent** -- compares predictions against lines to detect edges
- **bookie-breaker-prediction-engine** -- may use line movement as a feature
- **bookie-breaker-bookie-emulator** -- captures odds at bet placement, closing lines for CLV
- **bookie-breaker-cli** -- displays lines and line movement
- **bookie-breaker-ui** -- renders line movement charts and current odds
- **bookie-breaker-mcp-server** -- exposes line data as MCP tools

---

## Requirements

### Functional Requirements

- **FR-001:** Ingest betting lines from The Odds API every 60 seconds for all configured leagues during active game windows, and every 5 minutes during off-peak hours.
- **FR-002:** Receive real-time line updates from SharpAPI via SSE streaming with sub-second processing latency.
- **FR-003:** Normalize all ingested lines into the canonical BettingLine schema, standardizing team names, bet type labels, and odds formats (store American odds + implied probability).
- **FR-004:** Deduplicate incoming lines against the most recent stored snapshot, persisting only new or changed lines.
- **FR-005:** Retain every historical line snapshot indefinitely for full line movement tracking.
- **FR-006:** Identify and flag opening lines (first line posted for a game/market/sportsbook) and closing lines (final line before game start).
- **FR-007:** Support all bet types: spreads, totals, moneylines, player props, team props, game props, and futures across all 6 leagues (NFL, NBA, MLB, NCAA_FB, NCAA_BB, NCAA_BSB).
- **FR-008:** Serve current lines for any combination of game, bet type, and sportsbook via REST API.
- **FR-009:** Serve historical line movement data (ordered BettingLine snapshots) for any game/sportsbook/market combination, including computed LineMovement aggregates.
- **FR-010:** Serve closing lines for CLV calculations by bookie-emulator.
- **FR-011:** Detect and flag sharp line moves (significant moves in a short window) and steam moves when detectable from line movement patterns.
- **FR-012:** Publish a `lines.updated` event to Redis pub/sub channel `events:lines.updated` whenever new or changed lines are persisted, including league, affected game IDs, changed bet types, and change count.
- **FR-013:** Archive raw API responses permanently in a `raw_api_responses` TimescaleDB hypertable (source, timestamp, endpoint, HTTP status, response body). This data accumulates from day one for future LLM fine-tuning and training data. Additionally cache raw responses in Redis for 24 hours to support debugging and replay.
- **FR-014:** Track lines from 40+ sportsbooks as provided by The Odds API, maintaining a canonical Sportsbook registry with `is_sharp` flags for market-making books.

### Non-Functional Requirements

- **Latency:** Normalization + storage of a full poll cycle must complete in < 2 seconds. End-to-end from API response received to `lines.updated` event published must be < 5 seconds. API responses for current lines must return in < 200ms. Line movement queries must return in < 500ms.
- **Throughput:** Handle up to 50 poll cycles per hour across all leagues. Each poll cycle may return lines for up to 100 games with 40+ sportsbooks each (up to 4,000 line records per cycle). Serve up to 100 API requests/second from downstream consumers (agent, prediction-engine, bookie-emulator, interfaces).
- **Availability:** 99.5% uptime during active sports seasons. Graceful degradation: if external APIs are unavailable, serve cached/stored lines and log warnings. Resume polling automatically when APIs recover.
- **Storage:** Estimated 5-10 million BettingLine snapshots per year across all leagues (6 leagues x ~5,000 games/year x ~20 markets/game x ~10 snapshots/market). Growth rate: ~15,000-30,000 new rows/day during peak season. Raw API response cache: ~1-5 GB rolling 24-hour window.

### Data Ownership

This service is the source of truth for:

- **Sportsbook** -- canonical registry of tracked sportsbooks with keys, sharp flags, and active status.
- **BettingLine** -- all ingested line snapshots across all games, sportsbooks, and market types.
- **LineMovement** -- computed on read from BettingLine history; not stored separately.

### APIs Exposed

| Method + Path                          | Description                                                 | Key Query Parameters                        | Consumers                                               |
| -------------------------------------- | ----------------------------------------------------------- | ------------------------------------------- | ------------------------------------------------------- |
| `GET /api/v1/lines/{game_id}`          | Get current lines for a specific game                       | `market_type`, `sportsbook`, `side`         | agent, prediction-engine, bookie-emulator, CLI, UI, MCP |
| `GET /api/v1/lines`                    | Get current lines for multiple games                        | `league`, `game_ids`, `market_type`, `date` | agent, CLI, UI, MCP                                     |
| `GET /api/v1/lines/{game_id}/movement` | Get line movement history for a game                        | `sportsbook`, `market_type`, `selection`    | prediction-engine, agent, UI                            |
| `GET /api/v1/lines/{game_id}/closing`  | Get closing lines for a completed game                      | `sportsbook`, `market_type`                 | bookie-emulator (CLV calculation)                       |
| `GET /api/v1/lines/{game_id}/best`     | Get best available line across all sportsbooks for a market | `market_type`, `selection`                  | agent (edge detection)                                  |
| `GET /api/v1/sportsbooks`              | List all tracked sportsbooks                                | `is_sharp`, `is_active`                     | agent, prediction-engine                                |
| `GET /api/v1/lines/snapshot`           | Get a point-in-time snapshot of all lines for a game        | `game_id`, `timestamp`                      | bookie-emulator (placement odds capture)                |

### APIs Consumed

| Service                 | Endpoint                      | Purpose                                                |
| ----------------------- | ----------------------------- | ------------------------------------------------------ |
| The Odds API (external) | `GET /v4/sports/{sport}/odds` | Poll current betting lines across all sportsbooks      |
| SharpAPI (external)     | SSE stream endpoint           | Receive real-time line updates with sub-second latency |

### Events Published

| Event           | Channel                | Description                                                                                                                                                           |
| --------------- | ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `lines.updated` | `events:lines.updated` | Published when new or changed lines are detected and persisted. Payload includes league, affected game_ids, changed bet_types, sportsbooks updated, and change count. |

### Events Subscribed

None. The lines-service is a data producer only; it does not subscribe to any events.

### Storage Requirements

- **Database:** PostgreSQL (primary persistent store).
- **Key tables:**
  - `sportsbooks` -- canonical sportsbook registry (~50 rows, near-static).
  - `betting_lines` -- immutable line snapshots. Primary table, append-only. Estimated 5-10M rows/year.
  - `ingestion_log` -- tracks each poll cycle (timestamp, source, games fetched, lines upserted, errors). ~50K rows/year.
- **Estimated row counts:** Year 1: ~5-10M betting_lines rows. Year 3: ~20-30M rows.
- **Growth:** ~15,000-30,000 new betting_lines rows/day during peak multi-sport overlap (Oct-Mar).
- **Indexing priorities:**
  1. `betting_lines(game_id, market_type, sportsbook_id, timestamp DESC)` -- primary query pattern for current lines and line movement.
  2. `betting_lines(game_id, is_closing)` -- closing line lookups for CLV.
  3. `betting_lines(timestamp)` -- time-range queries for debugging and monitoring.
  4. `betting_lines(sportsbook_id, market_type)` -- sportsbook-level aggregation queries.
- **Redis:** Used for raw API response cache (24h TTL), recent event deduplication (1h TTL), and short-term line snapshot cache for frequently queried games (5m TTL).
