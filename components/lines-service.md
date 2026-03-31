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

| Source | Data | Mechanism |
|---|---|---|
| External odds APIs | Raw lines/odds across sportsbooks | Scheduled API polling |
| agent | Requests for specific lines, line movement, closing lines | API call |
| bookie-emulator | Requests for current odds at bet placement time, closing lines for CLV | API call |
| CLI / UI / MCP server | Line lookup and line movement requests | API call |

## Outputs

| Destination | Data | Mechanism |
|---|---|---|
| agent | Current lines, historical lines, line movement data | API response |
| prediction-engine | Market lines for edge calculation (predicted prob vs. implied prob) | API response |
| bookie-emulator | Current odds at placement time, closing lines | API response |
| CLI / UI | Lines, odds, line movement charts data | API response |

## Dependencies

- **External odds APIs** (external) -- source of all raw lines/odds data

## Dependents

- **bookie-breaker-agent** -- compares predictions against lines to detect edges
- **bookie-breaker-prediction-engine** -- may use line movement as a feature
- **bookie-breaker-bookie-emulator** -- captures odds at bet placement, closing lines for CLV
- **bookie-breaker-cli** -- displays lines and line movement
- **bookie-breaker-ui** -- renders line movement charts and current odds
- **bookie-breaker-mcp-server** -- exposes line data as MCP tools
