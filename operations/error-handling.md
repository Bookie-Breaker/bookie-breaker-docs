# Error Handling

How BookieBreaker detects, responds to, and recovers from failures across all 8 critical flows. Organized by failure category with specific detection and response strategies.

For the flows referenced here, see [Sequence Diagrams](../architecture/sequence-diagrams.md). For API error codes, see [API Design Principles](../api-contracts/README.md).

---

## 1. External API Failures

External dependencies: The Odds API (lines), SharpAPI SSE (live lines), nba_api / nfl_data_py / pybaseball (stats), Anthropic API (LLM analysis).

### Circuit Breaker Pattern

Each external API call is wrapped in a circuit breaker with three states:

| State                   | Behavior                                          | Transition                                                             |
| ----------------------- | ------------------------------------------------- | ---------------------------------------------------------------------- |
| **Closed** (normal)     | Requests pass through normally                    | Opens after **3 consecutive failures** or **5 failures in 60 seconds** |
| **Open** (tripped)      | Requests fail immediately without calling the API | Transitions to half-open after **60-second timeout**                   |
| **Half-open** (probing) | Allows **1 test request** through                 | Closes on success, reopens on failure                                  |

### Failure Modes by External API

#### The Odds API (lines-service)

| Failure                                     | Detection              | Response                                                                                                                                             |
| ------------------------------------------- | ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| API returns 5xx                             | HTTP status code check | Circuit breaker opens. Use last cached lines (Redis `stats:schedule:{league}:{date}`). Lines marked `is_stale=true` after 15 minutes without update. |
| API returns 429 (rate limited)              | HTTP 429 status        | Back off for `Retry-After` header duration. Reduce polling frequency to minimum (5-minute intervals). Log rate limit event.                          |
| API returns invalid/malformed JSON          | JSON parse error       | Discard response, log raw payload for debugging. Retry on next polling cycle. Do not update stored lines.                                            |
| API key expired or unauthorized (401/403)   | HTTP status code       | Circuit breaker opens. Alert operator immediately (this requires manual intervention). Pipeline continues with stale lines.                          |
| DNS resolution failure / connection timeout | Network-level error    | Circuit breaker counts as failure. Retry on next cycle.                                                                                              |

**Stale data policy:** Lines cached in PostgreSQL remain valid for edge detection for up to **30 minutes** during API outage. After 30 minutes, edges computed from those lines are flagged `is_stale=true` and excluded from auto-betting. Manual betting still possible with a stale warning.

#### SharpAPI SSE Stream (lines-service)

| Failure                       | Detection                                   | Response                                                                                                                     |
| ----------------------------- | ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| SSE connection drops          | Connection closed / heartbeat timeout (30s) | Reconnect with exponential backoff (1s, 2s, 4s, 8s, max 30s). Fall back to REST polling of The Odds API during reconnection. |
| SSE delivers malformed events | JSON parse error on event data              | Discard individual event, log for debugging. Stream stays connected for next event.                                          |
| SSE connection refused        | Connection error on initial connect         | Fall back entirely to REST polling. Log degradation. Retry SSE connection every 5 minutes.                                   |

#### nba_api / nfl_data_py / pybaseball (statistics-service)

| Failure                               | Detection                                    | Response                                                                                                                  |
| ------------------------------------- | -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| API rate limited (HTTP 429 or ban)    | HTTP status or connection refused            | Circuit breaker opens. Use Redis-cached stats (TTL 6 hours). Reduce polling frequency.                                    |
| API returns empty/partial data        | Response validation: expected fields missing | Log warning with details of missing fields. Return cached data if available. Flag stats as potentially incomplete.        |
| Python package throws exception       | Exception handler in ingestion code          | Log traceback. Skip this data source for current cycle. Retry on next scheduled ingestion. Other data sources unaffected. |
| External API schema change (breaking) | Field mapping errors, unexpected data types  | Log detailed error. Continue serving cached data. Requires code update to fix mapping -- alert operator.                  |

**Stale data policy:** Team/player stats cached in Redis with 6-hour TTL. If external APIs are down, stats remain usable for simulation and prediction. Stats that are 6+ hours old trigger a warning in simulation results but do not block the pipeline.

#### Anthropic API (agent)

| Failure                           | Detection                       | Response                                                                                                                                                                                          |
| --------------------------------- | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| API returns 5xx                   | HTTP status code                | Retry with exponential backoff (1s, 2s, 4s), max 3 retries. If all retries fail, return edge data without narrative analysis. Analysis is a "nice-to-have" -- edges and bets are the core output. |
| API returns 429 (rate limited)    | HTTP 429 status                 | Back off for `Retry-After` duration. Queue analysis for later generation. Return structured edge data immediately.                                                                                |
| API returns 400 (prompt too long) | HTTP 400 with token count error | Truncate context (remove lower-priority features like full game logs). Retry with shorter prompt.                                                                                                 |
| API timeout (>30s)                | Client-side timeout             | Return structured data without analysis. Cache timeout event. Retry analysis in background.                                                                                                       |
| API key invalid / billing issue   | HTTP 401/403                    | Alert operator. All analytical queries degrade to structured data only. Pipeline, edge detection, and paper betting continue unaffected.                                                          |

**Fallback:** When the Anthropic API is unavailable, the `POST /api/v1/agent/analysis` endpoint returns a structured summary of the data (simulation probabilities, key features, edge size) instead of a narrative. The `analysis.content` field contains a formatted data dump rather than LLM prose.

---

## 2. Inter-Service Failures

How services handle failures when calling other BookieBreaker services.

### Retry Policy

All inter-service REST calls use the same retry strategy:

| Parameter        | Value                                                        |
| ---------------- | ------------------------------------------------------------ |
| Max retries      | 3                                                            |
| Backoff strategy | Exponential: 500ms, 1s, 2s                                   |
| Retry on         | 5xx status codes, connection refused, timeout                |
| Do not retry on  | 4xx status codes (client errors are not transient)           |
| Idempotency      | All retried POST requests include `X-Idempotency-Key` header |

### Timeout Configuration

| Caller            | Callee             | Timeout    | Rationale                               |
| ----------------- | ------------------ | ---------- | --------------------------------------- |
| agent             | simulation-engine  | 5 minutes  | Monte Carlo simulation is CPU-intensive |
| agent             | prediction-engine  | 2 minutes  | ML inference + feature retrieval        |
| agent             | lines-service      | 5 seconds  | Simple data lookup                      |
| agent             | statistics-service | 5 seconds  | Simple data lookup                      |
| agent             | bookie-emulator    | 5 seconds  | Simple write/read                       |
| agent             | Anthropic API      | 30 seconds | LLM generation                          |
| simulation-engine | statistics-service | 5 seconds  | Feature/stats retrieval                 |
| prediction-engine | statistics-service | 5 seconds  | Feature retrieval                       |
| prediction-engine | lines-service      | 5 seconds  | Line movement retrieval                 |
| bookie-emulator   | lines-service      | 5 seconds  | Odds capture                            |
| bookie-emulator   | statistics-service | 5 seconds  | Game result retrieval                   |

### Graceful Degradation Matrix

What still works when a specific service is down:

| Service Down           | Still Works                                                                                      | Degraded                                                                                                                  | Broken                                                     |
| ---------------------- | ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| **lines-service**      | Stats queries, simulation (cached params), existing predictions                                  | Edge detection (uses stale lines, flagged), new paper bets (cannot capture odds)                                          | Fresh line queries, CLV calculation                        |
| **statistics-service** | Line queries, existing predictions, existing paper bets, LLM analysis (with cached context)      | Simulation (uses cached stats if <6h old), prediction (uses cached features)                                              | New simulations without cached stats, feature computation  |
| **simulation-engine**  | Line queries, stats queries, existing edges, paper bet grading, user queries about existing data | Full pipeline (cannot generate new simulations)                                                                           | New predictions for games without cached simulations       |
| **prediction-engine**  | Line queries, stats queries, simulation-only probabilities (no ML adjustment), paper bet grading | Edge detection (can fall back to raw simulation probabilities vs. market, with wider confidence intervals)                | Calibrated predictions, model retraining                   |
| **bookie-emulator**    | Full pipeline through edge detection, all queries except performance data                        | Edge alerts still fire, but no paper bets placed                                                                          | Paper bet placement, performance metrics, bankroll queries |
| **Redis**              | All sync REST calls (services talk directly), database reads/writes                              | Pub/sub events lost (polling fallbacks activate), cache misses (services fetch from source APIs directly, higher latency) | Real-time event delivery                                   |

### Failure Modes by Flow

#### Flow 1 - Lines Ingestion: statistics-service not involved, no inter-service calls

No inter-service dependencies. Failures are purely external API (see section 1).

#### Flow 2 - Full Pipeline: simulation-engine down

| Step          | Failure                                                  | Detection                           | Response                                                                                                                                                                                                                                |
| ------------- | -------------------------------------------------------- | ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Agent -> SimE | simulation-engine unreachable                            | Connection refused / 503            | Retry 3x with backoff. If all retries fail, check for cached simulation results in Redis (`sim:result:{game_id}:{config_hash}`). If cached results exist and are <2 hours old, proceed with those. If no cache, skip this game and log. |
| SimE -> SS    | statistics-service unreachable                           | Connection refused / 503 from stats | SimE returns 502 Bad Gateway to agent. Agent retries the full simulation request (idempotent). If stats-service stays down, games without cached features are skipped.                                                                  |
| Agent -> PE   | prediction-engine unreachable                            | Connection refused / 503            | Retry 3x. If PE is down, agent can fall back to raw simulation probabilities for edge detection (lower confidence, wider thresholds applied).                                                                                           |
| PE -> SS      | statistics-service unreachable during feature retrieval  | 502 from PE                         | PE uses cached features from Redis (`stats:features:{game_id}`, TTL 1 hour). If cache miss, PE returns 502 to agent. Agent skips ML adjustment for this game.                                                                           |
| PE -> LS      | lines-service unreachable during line movement retrieval | 502 from PE                         | PE proceeds without line movement features (sets them to neutral). Prediction quality slightly reduced but still usable.                                                                                                                |

#### Flow 3 - Paper Bet Placement: lines-service down

| Step     | Failure                                    | Detection                | Response                                                                                                                                                        |
| -------- | ------------------------------------------ | ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| BE -> LS | lines-service unreachable for odds capture | Connection refused / 503 | BE returns 502 to agent. Bet is NOT placed (odds cannot be verified). Agent logs the missed bet opportunity. Edge remains detected for potential manual review. |

#### Flow 4 - Paper Bet Grading: statistics-service down

| Step     | Failure                                        | Detection                | Response                                                                                                                           |
| -------- | ---------------------------------------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| BE -> SS | statistics-service unreachable for game result | Connection refused / 503 | Bet grading deferred. Polling fallback (every 30 minutes) will retry. Bet remains in PENDING state. No data loss.                  |
| BE -> LS | lines-service unreachable for closing line     | Connection refused / 503 | Bet graded for win/loss/push (game result available), but CLV calculation deferred. CLV updated later when lines-service recovers. |

#### Flow 5 & 6 - User Queries: any backend service down

| Step                 | Failure             | Detection                          | Response                                                                                                                                                                                                       |
| -------------------- | ------------------- | ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Agent -> any service | Service unreachable | Connection refused / 503 / timeout | Agent returns partial data from available services. Response includes a `warnings` field listing unavailable data sources. Example: `"warnings": ["simulation data unavailable - simulation-engine is down"]`. |

---

## 3. Data Quality Issues

### Stale Lines

| Issue                                                | Detection                                              | Response                                                                                                      |
| ---------------------------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| Lines not updated for >15 minutes during game window | Timestamp comparison: `now() - line.timestamp > 15min` | Flag lines as `is_stale=true`. Agent excludes stale lines from auto-betting. User queries show stale warning. |
| Lines not updated for >30 minutes                    | Timestamp comparison                                   | Remove affected edges from active list. Log stale line alert.                                                 |
| Opening lines missing (game just posted)             | Lines query returns empty for a scheduled game         | Skip edge detection for this game. Queue for next polling cycle.                                              |
| Line value seems wrong (e.g., spread of +/- 50)      | Range validation on normalized lines                   | Reject outlier lines (configurable thresholds per sport). Log rejected line with raw data for investigation.  |

### Missing Statistics

| Issue                                             | Detection                                                                                | Response                                                                                                           |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Feature vector incomplete (missing team stats)    | `GET /api/v1/stats/features/{game_id}` returns null fields                               | Simulation-engine skips game. Log warning: "Insufficient data for simulation of {game_id}".                        |
| Player injury data unavailable                    | Injury fields empty or stale (>24h old)                                                  | Prediction-engine sets injury impact features to 0 (neutral). Prediction confidence interval widened. Log warning. |
| Team stats for new/expansion team                 | Team not found in stats database                                                         | Skip games involving this team until stats are available. Log as data gap.                                         |
| Game result delayed (stats source slow to update) | `game.completed` event not received within expected window (2 hours after scheduled end) | Bookie-emulator polling fallback checks every 30 minutes. Bet remains PENDING. No incorrect grading occurs.        |

### Simulation Non-Convergence

| Issue                                                                         | Detection                                                    | Response                                                                                                                                                                 |
| ----------------------------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Distributions not converging within configured iterations                     | `converged: false` in simulation result after max iterations | If within 10% of convergence threshold, accept results with wider confidence intervals. If far from convergence, increase iterations to 2x and retry (up to 50,000 max). |
| Simulation produces degenerate distributions (all probability on one outcome) | Standard deviation of score distribution < 1.0               | Flag simulation as unreliable. Do not generate predictions from this result. Log for investigation (likely bad input parameters).                                        |
| Simulation runtime exceeds timeout (5 minutes)                                | Client-side timeout                                          | Return partial results if available (iterations completed so far). If <1000 iterations completed, treat as failure.                                                      |

---

## 4. Pipeline Failures

### Idempotent Pipeline Steps

Every step of the prediction pipeline is safe to re-run:

| Step                | Idempotency Mechanism                                                                                                                          |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Simulation          | `X-Idempotency-Key` header. Cached by `parameters_hash` -- same inputs produce same `simulation_run_id`. `force_refresh=true` overrides cache. |
| Prediction          | `X-Idempotency-Key` header. Same `game_id + simulation_run_id + market_types` returns cached prediction.                                       |
| Edge detection      | Pure computation (no side effects). Re-running with same predictions and lines produces same edges.                                            |
| Paper bet placement | `X-Idempotency-Key` header. Duplicate submission returns existing bet.                                                                         |

### Pipeline State Tracking

The agent tracks pipeline run state in its database:

```
pipeline_run:
  id: pipeline_run_id
  status: running | completed | failed | partial
  started_at: timestamp
  completed_at: timestamp
  steps:
    simulation: {status, game_ids_completed, game_ids_failed}
    prediction: {status, game_ids_completed, game_ids_failed}
    edge_detection: {status, edges_found}
    bet_placement: {status, bets_placed, bets_failed}
```

| Failure Scenario                                     | Detection                                   | Response                                                                                                                                                                                   |
| ---------------------------------------------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Pipeline crashes mid-execution                       | Process monitoring / health check           | Pipeline run marked `status: failed` with last completed step. Manual or scheduled re-trigger picks up from the beginning (idempotent -- completed steps return cached results instantly). |
| Partial batch failure (3 of 8 games fail simulation) | `failed_games > 0` in batch response        | Pipeline continues with successful games. Failed games logged. Pipeline status set to `partial`. Failed games can be retried independently.                                                |
| Pipeline triggered while another is running          | Check `pipeline_runs` for `status: running` | Reject duplicate run with `409 Conflict`. Return existing `pipeline_run_id` for status tracking.                                                                                           |
| Pipeline completes but produces zero edges           | `edges_found: 0`                            | Normal outcome (not a failure). Log for monitoring. No bets placed.                                                                                                                        |

### Manual Re-trigger

The agent exposes `POST /api/v1/agent/pipeline/run` for manual re-triggers:

- Set `force_refresh: true` to bypass all caches and re-simulate/re-predict from scratch.
- Set `game_ids` to retry specific failed games rather than the full batch.
- Idempotency keys ensure no duplicate bets even on re-trigger.

---

## 5. Storage Failures

### PostgreSQL Down

Each service that depends on PostgreSQL (lines-service, bookie-emulator, prediction-engine) handles database failures:

| Scenario                  | Detection                                      | Response                                                                                                                                            |
| ------------------------- | ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| Connection refused        | Database driver connection error               | Service health endpoint returns `"database": "unhealthy"`. Service returns `503 Service Unavailable` for all requests that require database access. |
| Query timeout             | Query exceeds configured timeout (default 10s) | Return `504 Gateway Timeout`. Log slow query for investigation.                                                                                     |
| Connection pool exhausted | All connections in use, new requests blocked   | Return `503 Service Unavailable`. Log pool exhaustion. Auto-recover when connections are returned to pool.                                          |
| Disk full                 | Write operations fail                          | Service returns `500 Internal Error`. Alert operator immediately. Read operations continue working.                                                 |

**Impact by service:**

| Service           | DB Down Impact                                                                                                                                               |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| lines-service     | Cannot store new line snapshots. Ingestion pauses but SSE/polling continues (data buffered in memory briefly). Existing lines still served from Redis cache. |
| bookie-emulator   | Cannot place new bets or grade bets. Existing bet data unavailable. Performance metrics unavailable.                                                         |
| prediction-engine | Cannot store new predictions or model versions. Cached predictions in Redis still served.                                                                    |

**Recovery:** Services automatically reconnect when PostgreSQL recovers. No data loss for events -- missed `game.completed` events are caught by the polling fallback. Missed line snapshots are not recoverable (those polling cycles are lost), but the next cycle captures current state.

### Redis Down

| Scenario                     | Detection                          | Response                                                                                                                      |
| ---------------------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| Connection refused           | Redis driver connection error      | Service health endpoint returns `"redis": "unhealthy"`.                                                                       |
| Pub/sub events undeliverable | Redis unavailable for PUBLISH      | Events are lost (fire-and-forget). All event-driven flows fall back to polling.                                               |
| Cache reads fail             | Redis GET returns connection error | Transparent fallback: services call source APIs directly instead of reading cache. Higher latency but functionally identical. |
| Cache writes fail            | Redis SET returns connection error | Silently skip cache write. Data still stored in PostgreSQL. Next request will miss cache and fetch from source.               |

**Polling fallbacks activated when Redis is down:**

| Event            | Polling Fallback                                                                   | Interval                    |
| ---------------- | ---------------------------------------------------------------------------------- | --------------------------- |
| `lines.updated`  | Agent's scheduled pipeline run fetches current lines regardless of events          | Per cron schedule (minutes) |
| `stats.updated`  | Agent's scheduled pipeline run fetches current stats regardless of events          | Per cron schedule (minutes) |
| `game.completed` | Bookie-emulator polls `GET /api/v1/stats/games?status=FINAL` for open bet game IDs | Every 30 minutes            |
| `edge.detected`  | CLI/UI polls `GET /api/v1/agent/edges` on a timer                                  | Every 5 minutes             |

**Recovery:** Redis reconnection is automatic. Cache repopulates organically as services make requests. No manual intervention needed.

---

## 6. Edge Cases

### Game Postponed or Cancelled After Bets Placed

| Scenario                                  | Detection                                                                    | Response                                                                                                                                                                      |
| ----------------------------------------- | ---------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Game postponed (rescheduled)              | statistics-service reports game status change to `POSTPONED`                 | Bookie-emulator marks bet as `VOID`. Stake returned to bankroll (no P&L impact). Edge marked stale. If game is rescheduled, a new edge detection cycle runs for the new date. |
| Game cancelled                            | statistics-service reports game status `CANCELLED`                           | Same as postponed: bet voided, stake returned.                                                                                                                                |
| Game goes to overtime                     | Game result includes `overtime: true`                                        | No special handling needed. Bet grading uses final score including overtime (standard practice).                                                                              |
| Game suspended mid-play and resumed later | Status transitions: `IN_PROGRESS` -> `SUSPENDED` -> `IN_PROGRESS` -> `FINAL` | Bookie-emulator waits for `FINAL` status. No premature grading.                                                                                                               |

### Line Moved Between Edge Detection and Paper Bet Placement

| Scenario                                      | Detection                                                                                                                               | Response                                                                                                                                                                                                                                                                                                                         |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Line moved in same direction (edge smaller)   | Bookie-emulator captures current odds at placement time. If `placement_odds != detection_odds`, the actual edge is recalculated.        | Bet placed at actual current odds. If recalculated edge falls below minimum threshold, bet is still placed (the threshold check happens at detection time; placement proceeds with current odds). The `edge_percentage` field on the bet reflects the detection-time edge, while `odds_american` reflects actual placement odds. |
| Line moved against us (edge gone or negative) | Time between edge detection and bet placement is typically <2 seconds (same pipeline run). Significant movement in that window is rare. | If this becomes a problem, add a pre-placement check: re-fetch current line and recompute edge before calling `POST /api/v1/emulator/bets`. Discard if edge is gone. Not implemented initially due to rarity.                                                                                                                    |
| Line removed entirely (market pulled)         | `GET /api/v1/lines/current` returns no line for this game/market/book                                                                   | Bookie-emulator returns `404` for odds capture. Bet not placed. Agent logs missed opportunity.                                                                                                                                                                                                                                   |

### Concurrent Pipeline Runs for the Same Game

| Scenario                                                         | Detection                                                     | Response                                                                                                                                                                                                    |
| ---------------------------------------------------------------- | ------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Two pipeline runs triggered simultaneously (e.g., cron + manual) | Agent checks for `status: running` pipeline before starting   | Second run rejected with `409 Conflict`. First run continues.                                                                                                                                               |
| Two pipeline runs for different leagues overlap                  | No conflict -- different games processed                      | Both runs proceed normally. No shared state between leagues.                                                                                                                                                |
| Event-triggered re-evaluation during scheduled pipeline          | `lines.updated` event arrives while pipeline is mid-execution | Agent queues the re-evaluation for after current pipeline completes. If the same game is affected, it will be re-evaluated with fresh lines in the queued run.                                              |
| Duplicate paper bet for same game/market                         | `X-Idempotency-Key` on `POST /api/v1/emulator/bets`           | Bookie-emulator returns existing bet (idempotent). Additionally, emulator enforces a unique constraint on `(game_id, market_type, selection)` to prevent duplicate positions regardless of idempotency key. |

---

## Summary Table: Flow x Failure Mode -> Response

| Flow                         | External API Failure                                                        | Inter-Service Failure                                                                                                                                        | Data Quality Issue                                                                                                  | Pipeline Failure                                                                   | Storage Failure                                                                                                      |
| ---------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| **1. Lines Ingestion**       | Circuit breaker on Odds API. Serve stale lines. SSE auto-reconnect.         | N/A (no inter-service calls)                                                                                                                                 | Reject malformed/outlier lines. Flag stale lines after 15 min.                                                      | N/A                                                                                | PostgreSQL down: buffer in memory, pause ingestion. Redis down: events lost, polling fallback.                       |
| **2. Full Pipeline**         | Anthropic API down: return structured data without narrative.               | Retry 3x with backoff. Skip games where simulation/prediction fails. Use cached results where available. Degrade to raw simulation probabilities if PE down. | Skip games with insufficient stats. Flag non-convergent simulations. Widen confidence intervals for stale features. | Idempotent re-run. Track partial completion. Failed games retryable independently. | PostgreSQL down: pipeline pauses (cannot store predictions). Redis down: cache misses increase latency, events lost. |
| **3. Paper Bet Placement**   | N/A                                                                         | Lines-service down: bet NOT placed (cannot capture odds). Log missed opportunity.                                                                            | Line moved: place at current odds. Line removed: skip bet.                                                          | Idempotency key prevents duplicate bets.                                           | PostgreSQL down: cannot place bets.                                                                                  |
| **4. Paper Bet Grading**     | N/A                                                                         | Stats-service down: defer grading, polling retry every 30 min. Lines-service down: grade win/loss but defer CLV.                                             | Game result delayed: bet stays PENDING, polling retries.                                                            | N/A                                                                                | PostgreSQL down: cannot update bet record. Grading deferred until recovery.                                          |
| **5. "Today's edges?"**      | N/A                                                                         | Any service down: return partial data with warnings.                                                                                                         | Stale edges flagged.                                                                                                | N/A                                                                                | Redis cache miss: fetch from services directly (higher latency).                                                     |
| **6. "Analyze Lakers game"** | Anthropic down: return structured data summary.                             | Any service down: return partial context with warnings. LLM analysis omits unavailable data.                                                                 | Missing data noted in analysis.                                                                                     | N/A                                                                                | Redis cache miss: fetch fresh data (higher latency).                                                                 |
| **7. Live Line Update**      | SharpAPI SSE drops: auto-reconnect with backoff. Fall back to REST polling. | PE down for existing predictions: cannot compute edge from new line. Queue for next pipeline run.                                                            | Outlier line rejected.                                                                                              | N/A                                                                                | Redis down: event lost. Agent picks up on next scheduled poll.                                                       |
| **8. Model Retraining**      | N/A                                                                         | Stats-service down: cannot fetch training data. Retrain aborted.                                                                                             | Insufficient training samples: abort with error.                                                                    | Shadow mode failures don't affect active model.                                    | PostgreSQL down: cannot store new model version.                                                                     |

---

## Monitoring and Alerting

Error handling is only effective if failures are visible. Key metrics to monitor:

| Metric                          | Source                               | Alert Threshold                                     |
| ------------------------------- | ------------------------------------ | --------------------------------------------------- |
| Circuit breaker state changes   | All services with external API calls | Any circuit breaker opens                           |
| Pipeline completion rate        | Agent pipeline status                | <90% games successfully processed                   |
| Stale line count                | lines-service                        | >10 games with stale lines during active window     |
| Bet grading backlog             | bookie-emulator                      | >5 bets PENDING for >2 hours after game completion  |
| Service health status           | All `/health` endpoints              | Any dependency reports `unhealthy`                  |
| Redis pub/sub subscriber count  | Redis `PUBSUB NUMSUB`                | Drops to 0 for any active channel                   |
| API error rate (5xx)            | All services                         | >5% of requests in 5-minute window                  |
| Simulation non-convergence rate | simulation-engine                    | >10% of simulations fail to converge                |
| Model prediction drift          | prediction-engine                    | Brier score degrades >10% over rolling 7-day window |

For implementation details on metrics collection and dashboards, see [Monitoring and Observability](monitoring-observability.md).
