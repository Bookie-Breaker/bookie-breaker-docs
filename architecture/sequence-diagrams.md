# Sequence Diagrams

Detailed Mermaid sequence diagrams for the 8 critical flows in BookieBreaker. Each diagram shows participants, HTTP calls with method and path, Redis pub/sub events, and conditional logic via alt/opt blocks.

For API conventions and endpoint details, see [API Design Principles](../api-contracts/README.md) and individual service API contracts.

---

## 1. Scheduled Lines Ingestion

Lines-service polls external odds APIs on a configurable schedule (1-5 minutes during game windows), normalizes data, stores snapshots, and publishes change notifications.

```mermaid
sequenceDiagram
    participant Cron as Scheduler (cron)
    participant LS as lines-service
    participant OddsAPI as The Odds API
    participant Sharp as SharpAPI (SSE)
    participant DB as Lines DB (PostgreSQL)
    participant Redis as Redis

    rect rgb(240, 248, 255)
        Note over Cron,LS: Polling loop (every 1-5 min)
        Cron->>LS: Timer fires

        par Poll REST source
            LS->>OddsAPI: GET /v4/sports/{sport}/odds
            OddsAPI-->>LS: 200 OK - Raw lines JSON
        and Listen to SSE stream
            Sharp-->>LS: SSE data frame (real-time push)
        end

        LS->>LS: Normalize to canonical format<br/>(team names, odds format, timestamps)
        LS->>LS: Deduplicate against latest stored snapshots<br/>(detect new, changed, removed lines)

        alt New or changed lines detected
            LS->>DB: INSERT new/changed line snapshots<br/>(TimescaleDB hypertable)
            LS->>Redis: PUBLISH events:lines.updated<br/>{league, game_ids, bet_types_changed, change_count}
        else No changes
            Note over LS: Skip storage and notification
        end
    end

    Note over Cron,LS: Loop repeats on next timer tick
```

---

## 2. Full Prediction Pipeline

The agent orchestrates the complete prediction pipeline: simulation, ML adjustment, edge detection, and optional paper bet placement.

```mermaid
sequenceDiagram
    participant Sched as Scheduler / lines.updated event
    participant Agent as agent
    participant SS as statistics-service
    participant SimE as simulation-engine
    participant PE as prediction-engine
    participant LS as lines-service
    participant BE as bookie-emulator
    participant LLM as Anthropic API
    participant Redis as Redis

    Sched->>Agent: Trigger pipeline<br/>(scheduled cron or lines.updated event)
    Agent->>Agent: Determine games needing predictions<br/>(upcoming games, changed lines/stats)

    %% Step 1: Simulation
    Agent->>SimE: POST /api/v1/sim/simulations/batch<br/>{games: [{game_id, config}], default_config}

    loop For each game in batch
        SimE->>SS: GET /api/v1/stats/features/{game_id}
        SS-->>SimE: 200 OK - Feature vector<br/>(team ratings, injuries, rest days)
        SimE->>SS: GET /api/v1/stats/teams/{team_id}/stats
        SS-->>SimE: 200 OK - Detailed team stats
        SimE->>SimE: Run Monte Carlo simulation<br/>(10,000-50,000 iterations)
    end

    SimE-->>Agent: 201 Created - Batch results<br/>{batch_id, results: [{simulation_run_id, result}]}
    SimE->>Redis: PUBLISH events:simulation.completed<br/>{batch_id, game_ids, iterations_per_game}

    %% Step 2: Prediction (ML adjustment)
    loop For each completed simulation
        Agent->>PE: POST /api/v1/predict/predictions<br/>{game_id, simulation_run_id, market_types}

        PE->>SS: GET /api/v1/stats/features/{game_id}
        SS-->>PE: 200 OK - Contextual features<br/>(injuries, rest, travel)
        PE->>LS: GET /api/v1/lines/game/{game_id}/movement
        LS-->>PE: 200 OK - Line movement data
        PE->>PE: Apply gradient boosting ML adjustments<br/>(XGBoost model)

        PE-->>Agent: 201 Created - Calibrated predictions<br/>{predictions: [{predicted_probability, confidence}]}
    end

    %% Step 3: Edge detection
    Agent->>LS: GET /api/v1/lines/game/{game_id}/best
    LS-->>Agent: 200 OK - Best available lines per market

    Agent->>Agent: Compare predicted probabilities vs.<br/>implied probabilities from best lines
    Agent->>Agent: Compute edge = predicted_prob - implied_prob<br/>Filter edges above threshold (e.g., 3%)

    opt Edges found above threshold
        Agent->>Redis: PUBLISH events:edge.detected<br/>{game_id, market_type, edge_pct, predicted_prob}

        %% Step 4: Paper bet placement
        loop For each qualifying edge
            Agent->>BE: POST /api/v1/emulator/bets<br/>{game_id, edge_id, market_type, selection,<br/>predicted_probability, edge_percentage, stake}<br/>X-Idempotency-Key: {uuid}
            BE->>LS: GET /api/v1/lines/current<br/>?game_id={game_id}&market_type={type}
            LS-->>BE: 200 OK - Current odds at placement
            BE-->>Agent: 201 Created - Bet confirmation<br/>{bet_id, odds_american, stake}
        end

        %% Step 5: LLM analysis (optional)
        Agent->>LLM: POST /v1/messages<br/>{model, system, messages with edge context}
        LLM-->>Agent: 200 OK - Natural language analysis
        Agent->>Agent: Cache analysis, alert interfaces
    end
```

---

## 3. Paper Bet Placement

When the agent detects an edge that meets threshold criteria, it places a paper (virtual) bet through the bookie-emulator.

```mermaid
sequenceDiagram
    participant Agent as agent
    participant BE as bookie-emulator
    participant LS as lines-service
    participant DB as Emulator DB (PostgreSQL)

    Note over Agent: Edge detected:<br/>game_id, market_type, selection,<br/>predicted_prob, edge_pct

    Agent->>Agent: Calculate stake via Kelly criterion<br/>(kelly_fraction * edge / (odds - 1))

    Agent->>BE: POST /api/v1/emulator/bets<br/>{game_id, edge_id, market_type, selection,<br/>side, predicted_probability, edge_percentage,<br/>stake, reasoning}<br/>X-Idempotency-Key: {uuid}

    BE->>BE: Validate request<br/>(game not started, stake within limits,<br/>daily exposure not exceeded)

    alt Validation passes
        BE->>LS: GET /api/v1/lines/current<br/>?game_id={game_id}&market_type={type}&sportsbook={book}
        LS-->>BE: 200 OK - Current line with odds

        BE->>DB: INSERT paper bet<br/>{game_id, edge_id, selection, odds_american,<br/>odds_decimal, stake, result: PENDING}

        BE-->>Agent: 201 Created<br/>{bet_id, odds_american, odds_decimal,<br/>stake, placed_at}

    else Game already started
        BE-->>Agent: 422 Unprocessable Entity<br/>{code: UNPROCESSABLE_ENTITY,<br/>message: "Game has already started"}

    else Stake exceeds limits
        BE-->>Agent: 422 Unprocessable Entity<br/>{code: UNPROCESSABLE_ENTITY,<br/>message: "Stake exceeds max_bet_units"}

    else Idempotency key already exists
        BE-->>Agent: 200 OK<br/>(returns existing bet)
    end
```

---

## 4. Paper Bet Grading

After a game completes, bookie-emulator grades open paper bets by comparing the game result against the bet terms and computes Closing Line Value.

```mermaid
sequenceDiagram
    participant SS as statistics-service
    participant Redis as Redis
    participant BE as bookie-emulator
    participant LS as lines-service
    participant DB as Emulator DB (PostgreSQL)

    SS->>Redis: PUBLISH events:game.completed<br/>{game_id, home_team, away_team,<br/>home_score, away_score, league}

    Redis->>BE: Deliver game.completed event

    BE->>BE: Check for open bets on this game_id

    alt Open bets exist for this game
        BE->>SS: GET /api/v1/stats/games/{game_id}
        SS-->>BE: 200 OK - Final game result<br/>{home_score, away_score, total, margin}

        BE->>LS: GET /api/v1/lines/game/{game_id}/closing
        LS-->>BE: 200 OK - Closing lines<br/>{line_value, odds_american per sportsbook}

        loop For each open bet on this game
            BE->>BE: Grade bet against game result<br/>(evaluate spread cover, total over/under,<br/>moneyline winner)

            alt Spread bet
                BE->>BE: actual_margin vs. line_value<br/>WIN if margin > spread, LOSS if <, PUSH if =
            else Total bet
                BE->>BE: actual_total vs. line_value<br/>WIN if over, LOSS if under, PUSH if =
            else Moneyline bet
                BE->>BE: Winner matches selection?<br/>WIN or LOSS
            end

            BE->>BE: Calculate profit/loss<br/>(WIN: stake * (decimal_odds - 1)<br/>LOSS: -stake, PUSH: 0)

            BE->>BE: Calculate CLV<br/>(placement_implied_prob - closing_implied_prob)

            BE->>DB: UPDATE bet SET result, profit_loss,<br/>closing_line_value, closing_odds, clv,<br/>graded_at = now()
        end

        BE->>BE: Recompute bankroll snapshot<br/>(total P&L, ROI, win_rate, avg_clv)
        BE->>DB: UPDATE bankroll and performance aggregates

    else No open bets for this game
        Note over BE: No action needed
    end

    Note over BE: Fallback: polling loop every 30 min<br/>checks statistics-service for completed<br/>games with open bets (handles missed events)
```

---

## 5. User Query: "What are today's edges?"

A user requests current edges through any interface. The agent queries prediction-engine and lines-service to return an enriched edge list.

```mermaid
sequenceDiagram
    participant User
    participant UI as CLI / UI / MCP
    participant Agent as agent
    participant PE as prediction-engine
    participant LS as lines-service
    participant Cache as Redis Cache

    User->>UI: "What are today's edges?"

    UI->>Agent: GET /api/v1/agent/edges<br/>?date=2026-03-30&is_stale=false

    Agent->>Cache: GET agent:dashboard:{league}
    alt Cache hit (< 5 min old)
        Cache-->>Agent: Cached edge summary
    else Cache miss or stale
        par Fetch predictions
            Agent->>PE: GET /api/v1/predict/games/{game_id}/edges<br/>?min_edge=0
            PE-->>Agent: 200 OK - Edges per game<br/>{edges: [{prediction_id, market_type,<br/>predicted_probability, implied_probability,<br/>edge_percentage}]}
        and Fetch current lines
            Agent->>LS: GET /api/v1/lines/current<br/>?date=2026-03-30
            LS-->>Agent: 200 OK - Current best lines<br/>{sportsbook_key, odds_american, line_value}
        end

        Agent->>Agent: Enrich edges with line data<br/>(best odds, sportsbook, kelly stake)
        Agent->>Agent: Sort by edge_percentage descending
        Agent->>Agent: Flag stale edges<br/>(lines older than threshold)
        Agent->>Cache: SETEX agent:dashboard:{league} 300 {data}
    end

    Agent-->>UI: 200 OK<br/>{data: [{game, market_type, selection,<br/>edge_percentage, odds_american,<br/>sportsbook_key, recommended_stake}]}

    UI-->>User: Render formatted edge list<br/>(table in CLI, cards in UI,<br/>tool result in MCP)
```

---

## 6. User Query: "Analyze the Lakers game"

A user requests a deep analysis of a specific game. The agent gathers data from multiple services and uses the Anthropic LLM API to generate a narrative.

```mermaid
sequenceDiagram
    participant User
    participant UI as CLI / UI / MCP
    participant Agent as agent
    participant SimE as simulation-engine
    participant PE as prediction-engine
    participant LS as lines-service
    participant BE as bookie-emulator
    participant SS as statistics-service
    participant LLM as Anthropic API
    participant Cache as Redis Cache

    User->>UI: "Analyze the Lakers game"
    UI->>Agent: POST /api/v1/agent/analysis<br/>{analysis_type: "GAME_PREVIEW",<br/>game_id: "{lakers_game_id}"}

    Agent->>Cache: GET agent:analysis:{game_id}
    alt Cache hit (< 1 hour old)
        Cache-->>Agent: Cached analysis text
    else Cache miss or stale
        par Gather context from all services
            Agent->>SimE: GET /api/v1/sim/games/{game_id}/latest
            SimE-->>Agent: 200 OK - Simulation results<br/>{home_win_prob, mean_total, mean_margin,<br/>spread_cover_probabilities, percentiles}
        and
            Agent->>PE: GET /api/v1/predict/games/{game_id}/latest
            PE-->>Agent: 200 OK - Calibrated predictions<br/>{predictions: [{market_type, predicted_prob,<br/>adjustment_magnitude, feature_importance}]}
        and
            Agent->>LS: GET /api/v1/lines/game/{game_id}
            LS-->>Agent: 200 OK - Current lines across books
            Agent->>LS: GET /api/v1/lines/game/{game_id}/movement
            LS-->>Agent: 200 OK - Line movement history
        and
            Agent->>BE: GET /api/v1/emulator/bets<br/>?game_id={game_id}&status=all
            BE-->>Agent: 200 OK - Paper bets on this game<br/>(if any)
        and
            Agent->>SS: GET /api/v1/stats/features/{game_id}
            SS-->>Agent: 200 OK - Feature vector<br/>(injuries, rest, matchup context)
        end

        Agent->>Agent: Assemble LLM prompt with all context:<br/>simulation distributions, predictions,<br/>current lines, movement, injuries,<br/>paper bets, feature importance

        Agent->>LLM: POST /v1/messages<br/>{model: "claude-sonnet-4-20250514",<br/>system: "Sports analyst prompt",<br/>messages: [{role: "user", content: assembled_context}]}
        LLM-->>Agent: 200 OK - Narrative analysis<br/>(markdown formatted)

        Agent->>Cache: SETEX agent:analysis:{game_id} 3600 {analysis}
    end

    Agent-->>UI: 201 Created<br/>{data: {title, content, model_used,<br/>input_summary, created_at}}
    UI-->>User: Render analysis<br/>(markdown in CLI, rich text in UI)
```

---

## 7. Live Line Update

When SharpAPI pushes a real-time line update, lines-service stores it and the agent checks whether the new line creates or invalidates an edge.

```mermaid
sequenceDiagram
    participant Sharp as SharpAPI (SSE)
    participant LS as lines-service
    participant DB as Lines DB (PostgreSQL)
    participant Redis as Redis
    participant Agent as agent
    participant PE as prediction-engine

    Sharp-->>LS: SSE data frame<br/>(new line for game_id, sportsbook, market_type)

    LS->>LS: Normalize incoming line<br/>(canonical format)
    LS->>LS: Compare against latest stored snapshot
    LS->>DB: INSERT line snapshot<br/>(timestamped, full history retained)

    LS->>Redis: PUBLISH events:lines.updated<br/>{league, game_ids, bet_types_changed,<br/>is_live: true, source: "sharpapi"}

    Redis->>Agent: Deliver lines.updated (is_live=true)

    Agent->>Agent: Check if existing prediction covers<br/>affected game_id and market_type

    alt Existing prediction found
        Agent->>PE: GET /api/v1/predict/games/{game_id}/latest
        PE-->>Agent: 200 OK - Latest prediction<br/>{predicted_probability}

        Agent->>Agent: Compute new implied probability from<br/>updated line odds
        Agent->>Agent: Calculate edge =<br/>predicted_prob - new_implied_prob

        alt New edge above threshold
            Agent->>Redis: PUBLISH events:edge.detected<br/>{game_id, market_type, edge_pct,<br/>predicted_prob, implied_prob, odds_american}

            Note over Agent: Alert user via subscribed interfaces<br/>(CLI watch mode, UI real-time, MCP)

        else Edge below threshold or negative
            Note over Agent: No action - log for monitoring
        end

        alt Previous edge now invalidated by line move
            Agent->>Agent: Mark previous edge as stale
        end

    else No existing prediction
        Note over Agent: Queue game for next pipeline run
    end
```

---

## 8. Model Retraining

Triggered manually or when performance metrics indicate degradation. A new model version is trained, validated in shadow mode, then promoted or discarded.

```mermaid
sequenceDiagram
    participant Trigger as Manual / Performance Monitor
    participant Agent as agent
    participant PE as prediction-engine
    participant SS as statistics-service
    participant DB as Prediction DB (PostgreSQL)

    Trigger->>Agent: Retrain request<br/>(performance degradation detected<br/>or manual trigger)

    Agent->>PE: POST /api/v1/predict/models/retrain<br/>{sport: "BASKETBALL", market_type: "SPREAD",<br/>training_config: {min_samples: 1000,<br/>test_split: 0.2, date_from, date_to}}

    PE-->>Agent: 202 Accepted<br/>{retrain_id, status: "started",<br/>estimated_duration_minutes: 15}

    rect rgb(240, 248, 255)
        Note over PE: Training process (async)

        PE->>SS: GET /api/v1/stats/games<br/>?league=NBA&status=FINAL<br/>&date_from={date_from}&date_to={date_to}
        SS-->>PE: 200 OK - Historical games list

        loop For each historical game
            PE->>SS: GET /api/v1/stats/features/{game_id}
            SS-->>PE: 200 OK - Historical feature vectors
        end

        PE->>PE: Build training dataset<br/>(features + actual outcomes)
        PE->>PE: Split into train/test sets
        PE->>PE: Train new XGBoost model
        PE->>PE: Evaluate on test set<br/>(Brier score, log loss, calibration error,<br/>backtest ROI)
        PE->>DB: INSERT new model version<br/>{version_tag, algorithm, evaluation_metrics,<br/>is_active: false}
    end

    rect rgb(255, 248, 240)
        Note over PE: Shadow mode validation

        PE->>PE: Deploy new model in shadow mode<br/>(runs alongside active model,<br/>predictions logged but not used)

        loop For each new game during validation period
            PE->>PE: Generate predictions with active model
            PE->>PE: Generate predictions with shadow model
            PE->>DB: Log both predictions for comparison
        end

        PE->>PE: After validation period (e.g., 1-2 weeks),<br/>compare shadow vs. active model performance
    end

    alt Shadow model outperforms active model
        PE->>DB: UPDATE active model SET is_active = false
        PE->>DB: UPDATE shadow model SET is_active = true
        Note over PE: New model promoted to production
    else Shadow model underperforms
        PE->>DB: DELETE or archive shadow model
        Note over PE: Shadow model discarded,<br/>active model remains
    end

    Agent->>PE: GET /api/v1/predict/models?is_active=true
    PE-->>Agent: 200 OK - Current active models
    Note over Agent: Log retraining outcome for monitoring
```
