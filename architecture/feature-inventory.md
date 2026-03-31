# BookieBreaker Feature Inventory

Master inventory of all features across the BookieBreaker system. Each feature has a short ID, one-line description, owning service(s), and priority level.

**Priority levels:**
- **P0** -- MVP. Required for the system to function and deliver value.
- **P1** -- Important. Needed for production-quality experience but not day-one blockers.
- **P2** -- Nice to have. Enhances the system but can be deferred.

**Supported leagues:** NFL, NBA, MLB, NCAA Football, NCAA Basketball, NCAA Baseball.

---

## 1. Core Pipeline Features

Features related to the main data-to-prediction pipeline: data ingestion, simulation, prediction, and edge detection.

### Data Ingestion

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| PIPE-001 | Ingest team and player statistics from external sports data APIs on a scheduled cadence | statistics-service | P0 |
| PIPE-002 | Normalize raw statistics into canonical format across different source APIs and sports | statistics-service | P0 |
| PIPE-003 | Store historical and current statistics at game-level, season-level, and career-level granularity | statistics-service | P0 |
| PIPE-004 | Compute derived statistics and advanced metrics (rolling averages, per-game rates, offensive/defensive ratings, pace, efficiency) | statistics-service | P0 |
| PIPE-005 | Ingest injury reports and roster information from source APIs | statistics-service | P0 |
| PIPE-006 | Ingest game schedules across all 6 leagues | statistics-service | P0 |
| PIPE-007 | Ingest and store final game results (scores, box scores) for bet grading and model evaluation | statistics-service | P0 |
| PIPE-008 | Ingest betting lines and odds from external odds APIs on a regular polling schedule | lines-service | P0 |
| PIPE-009 | Normalize lines into canonical format across different source APIs and sportsbooks | lines-service | P0 |
| PIPE-010 | Store historical lines with timestamps for line movement tracking | lines-service | P0 |
| PIPE-011 | Serve current lines/odds for any supported game, bet type, and sportsbook | lines-service | P0 |
| PIPE-012 | Track and serve historical line movement data (open to current) | lines-service | P1 |
| PIPE-013 | Provide closing lines (final line before game start) for CLV calculations | lines-service | P0 |
| PIPE-014 | Detect and flag sharp line moves and steam moves | lines-service | P2 |

### Simulation

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| PIPE-020 | Sport-agnostic Monte Carlo simulation framework with configurable iteration count and convergence criteria | simulation-engine | P0 |
| PIPE-021 | Football simulation plugin: discrete play-by-play or drive-level simulation for NFL and NCAA Football | simulation-engine | P0 |
| PIPE-022 | Basketball simulation plugin: continuous flow / possession-based simulation for NBA and NCAA Basketball | simulation-engine | P0 |
| PIPE-023 | Baseball simulation plugin: pitcher-batter matchup simulation with plate appearance resolution for MLB and NCAA Baseball | simulation-engine | P0 |
| PIPE-024 | Produce full score distributions, margin distributions, and total distributions from simulations | simulation-engine | P0 |
| PIPE-025 | Produce derived prop-relevant distributions from game simulations (player stat distributions, team stat distributions) | simulation-engine | P1 |
| PIPE-026 | Provide simulation metadata: convergence diagnostics, variance estimates, distribution shapes | simulation-engine | P1 |
| PIPE-027 | Support reproducible simulations via random seed management | simulation-engine | P1 |

### Prediction (ML Adjustment)

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| PIPE-030 | Apply ML-based adjustments to simulation output distributions for contextual factors | prediction-engine | P0 |
| PIPE-031 | Sport-specific feature engineering: rest days, travel distance, injury impact scores, weather effects | prediction-engine | P0 |
| PIPE-032 | Train and maintain gradient boosting models as baseline prediction models | prediction-engine | P0 |
| PIPE-033 | Produce calibrated probability outputs (not just point predictions) with confidence intervals | prediction-engine | P0 |
| PIPE-034 | Track model evaluation metrics: calibration curves, Brier scores, log loss over time | prediction-engine | P1 |
| PIPE-035 | Support model versioning for tracking prediction quality across model iterations | prediction-engine | P1 |
| PIPE-036 | Support A/B testing of model variants | prediction-engine | P2 |
| PIPE-037 | Provide feature importance rankings for each prediction | prediction-engine | P1 |
| PIPE-038 | Incorporate line movement patterns and public betting percentages as ML features | prediction-engine | P1 |
| PIPE-039 | Support sport-specific model variants (separate models per league/sport) | prediction-engine | P1 |

### Edge Detection

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| PIPE-040 | Compare calibrated probabilities against market-implied probabilities to identify +EV edges | agent | P0 |
| PIPE-041 | Quantify edge size (predicted probability minus implied probability) for each identified edge | agent | P0 |
| PIPE-042 | Filter edges by configurable minimum edge threshold | agent | P0 |
| PIPE-043 | Rank edges by expected value, confidence, and edge size | agent | P1 |
| PIPE-044 | Alert when new +EV edges are detected | agent | P0 |

### Pipeline Orchestration

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| PIPE-050 | Orchestrate end-to-end pipeline: stats pull -> simulation -> prediction -> edge detection -> alerts | agent | P0 |
| PIPE-051 | Schedule pipeline runs on configurable cadence (daily, pre-game, etc.) | agent | P0 |
| PIPE-052 | Coordinate service calls in correct dependency order | agent | P0 |
| PIPE-053 | Report pipeline status (running, completed, failed) to all interfaces | agent | P1 |
| PIPE-054 | Support on-demand pipeline runs triggered by user | agent | P1 |

---

## 2. Bet Type Features

Features for each supported bet type across all 6 leagues. Each requires simulation output, ML-calibrated probabilities, line comparison, and edge detection.

### Spreads

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| BET-001 | Simulate game margin distributions to produce spread probabilities for all 6 leagues | simulation-engine, prediction-engine | P0 |
| BET-002 | Ingest and serve current spread lines and odds across sportsbooks | lines-service | P0 |
| BET-003 | Detect +EV edges on spread bets by comparing predicted cover probability to implied probability | agent, prediction-engine | P0 |
| BET-004 | Support alternate spread lines (e.g., buying/selling points) | lines-service, simulation-engine | P1 |
| BET-005 | Track spread bet performance in paper trading with win rate and CLV | bookie-emulator | P0 |

### Totals (Over/Under)

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| BET-010 | Simulate combined score distributions to produce total probabilities for all 6 leagues | simulation-engine, prediction-engine | P0 |
| BET-011 | Ingest and serve current total lines and odds across sportsbooks | lines-service | P0 |
| BET-012 | Detect +EV edges on total bets by comparing predicted over/under probability to implied probability | agent, prediction-engine | P0 |
| BET-013 | Support alternate total lines | lines-service, simulation-engine | P1 |
| BET-014 | Apply weather adjustments to total predictions (outdoor sports: NFL, NCAA FB, MLB, NCAA Baseball) | prediction-engine | P1 |
| BET-015 | Track total bet performance in paper trading | bookie-emulator | P0 |

### Moneylines

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| BET-020 | Produce win probability for each team from simulation distributions for all 6 leagues | simulation-engine, prediction-engine | P0 |
| BET-021 | Ingest and serve current moneyline odds across sportsbooks | lines-service | P0 |
| BET-022 | Detect +EV edges on moneyline bets by comparing predicted win probability to implied probability | agent, prediction-engine | P0 |
| BET-023 | Track moneyline bet performance in paper trading | bookie-emulator | P0 |

### Player Props

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| BET-030 | Simulate individual player stat distributions from game simulations (e.g., passing yards, points scored, strikeouts) | simulation-engine | P1 |
| BET-031 | NFL player props: passing yards, rushing yards, receiving yards, touchdowns, completions, interceptions, receptions | simulation-engine, prediction-engine | P1 |
| BET-032 | NBA player props: points, rebounds, assists, three-pointers, steals, blocks, points+rebounds+assists combos | simulation-engine, prediction-engine | P1 |
| BET-033 | MLB player props: hits, RBIs, home runs, strikeouts (pitcher), total bases, runs scored | simulation-engine, prediction-engine | P1 |
| BET-034 | NCAA Football player props: passing yards, rushing yards, receiving yards, touchdowns | simulation-engine, prediction-engine | P1 |
| BET-035 | NCAA Basketball player props: points, rebounds, assists | simulation-engine, prediction-engine | P1 |
| BET-036 | NCAA Baseball player props: hits, RBIs, strikeouts (pitcher) | simulation-engine, prediction-engine | P2 |
| BET-037 | Ingest and serve current player prop lines and odds | lines-service | P1 |
| BET-038 | Detect +EV edges on player prop bets | agent, prediction-engine | P1 |
| BET-039 | Track player prop bet performance in paper trading | bookie-emulator | P1 |

### Team Props

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| BET-040 | Simulate team-level stat distributions (e.g., team total points, team total yards, team total hits) | simulation-engine | P1 |
| BET-041 | NFL team props: team total points, team total yards, team total touchdowns | simulation-engine, prediction-engine | P1 |
| BET-042 | NBA team props: team total points, team total rebounds, team total three-pointers | simulation-engine, prediction-engine | P1 |
| BET-043 | MLB team props: team total runs, team total hits, team total errors | simulation-engine, prediction-engine | P1 |
| BET-044 | NCAA Football team props: team total points, team total yards | simulation-engine, prediction-engine | P2 |
| BET-045 | NCAA Basketball team props: team total points | simulation-engine, prediction-engine | P2 |
| BET-046 | NCAA Baseball team props: team total runs, team total hits | simulation-engine, prediction-engine | P2 |
| BET-047 | Ingest and serve current team prop lines and odds | lines-service | P1 |
| BET-048 | Detect +EV edges on team prop bets | agent, prediction-engine | P1 |
| BET-049 | Track team prop bet performance in paper trading | bookie-emulator | P1 |

### Game Props

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| BET-050 | Simulate game-level prop outcomes (e.g., first team to score, highest scoring half/quarter, will there be overtime) | simulation-engine | P2 |
| BET-051 | NFL game props: first team to score, highest scoring quarter, overtime yes/no, margin of victory band | simulation-engine, prediction-engine | P2 |
| BET-052 | NBA game props: first team to score, highest scoring quarter, overtime yes/no, margin of victory band | simulation-engine, prediction-engine | P2 |
| BET-053 | MLB game props: first team to score, will there be extra innings, run in first inning yes/no | simulation-engine, prediction-engine | P2 |
| BET-054 | Ingest and serve current game prop lines and odds | lines-service | P2 |
| BET-055 | Detect +EV edges on game prop bets | agent, prediction-engine | P2 |
| BET-056 | Track game prop bet performance in paper trading | bookie-emulator | P2 |

### Futures

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| BET-060 | Model season-level outcomes: division winners, conference winners, championship winners | simulation-engine, prediction-engine | P2 |
| BET-061 | Model season-level player awards (MVP, ROY, etc.) | simulation-engine, prediction-engine | P2 |
| BET-062 | Model season win totals for each team | simulation-engine, prediction-engine | P2 |
| BET-063 | Ingest and serve current futures odds | lines-service | P2 |
| BET-064 | Detect +EV edges on futures bets | agent, prediction-engine | P2 |
| BET-065 | Track futures bet performance in paper trading | bookie-emulator | P2 |

### Live / In-Game Betting

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| BET-070 | Support real-time simulation updates during live games using current game state | simulation-engine | P2 |
| BET-071 | Ingest live in-game lines and odds | lines-service | P2 |
| BET-072 | Detect +EV edges on live bets by comparing updated simulation probabilities to live lines | agent, prediction-engine | P2 |
| BET-073 | Alert on live edges with time sensitivity | agent | P2 |
| BET-074 | Track live bet performance in paper trading | bookie-emulator | P2 |

### Parlays (Correlated Legs)

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| BET-080 | Model correlation between bet legs from the same game (e.g., spread + total, player prop + team total) | simulation-engine, prediction-engine | P2 |
| BET-081 | Calculate true parlay probability accounting for leg correlations (not naive independent multiplication) | prediction-engine | P2 |
| BET-082 | Detect +EV edges on correlated parlays where sportsbooks misprice correlation | agent, prediction-engine | P2 |
| BET-083 | Support same-game parlay (SGP) correlation modeling | simulation-engine, prediction-engine | P2 |
| BET-084 | Track parlay bet performance in paper trading | bookie-emulator | P2 |

---

## 3. Paper Trading Features

Features for the bookie emulator: virtual bet placement, tracking, grading, and performance analytics.

### Bet Placement and Management

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| PAPER-001 | Place virtual bets with exact line/odds at time of placement, stake, and bet type | bookie-emulator | P0 |
| PAPER-002 | Persist all open and historical paper bets | bookie-emulator | P0 |
| PAPER-003 | Support all bet types for paper trading: spreads, totals, moneylines, player props, team props, game props, futures, live, parlays | bookie-emulator | P0 |
| PAPER-004 | Record edge size and predicted probability at time of bet placement | bookie-emulator | P0 |
| PAPER-005 | Automatically place paper bets on detected edges when configured | agent, bookie-emulator | P1 |

### Bet Grading

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| PAPER-010 | Grade bets as win/loss/push when games complete using final results from statistics-service | bookie-emulator, statistics-service | P0 |
| PAPER-011 | Handle push scenarios correctly (return of stake) | bookie-emulator | P0 |
| PAPER-012 | Grade player prop bets using individual player box score data | bookie-emulator, statistics-service | P1 |
| PAPER-013 | Grade parlay bets (all legs must win) | bookie-emulator | P2 |

### Performance Analytics

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| PAPER-020 | Compute overall ROI (return on investment) across all paper bets | bookie-emulator | P0 |
| PAPER-021 | Compute win rate across all paper bets | bookie-emulator | P0 |
| PAPER-022 | Compute closing line value (CLV) by comparing bet placement odds to closing odds | bookie-emulator, lines-service | P0 |
| PAPER-023 | Compute units won/lost | bookie-emulator | P0 |
| PAPER-024 | Generate calibration curves (predicted probability vs. actual outcome frequency) | bookie-emulator | P1 |
| PAPER-025 | Track performance over configurable time windows: daily, weekly, monthly, all-time | bookie-emulator | P1 |
| PAPER-026 | Break down performance by sport/league | bookie-emulator | P1 |
| PAPER-027 | Break down performance by bet type (spread, total, moneyline, props, etc.) | bookie-emulator | P1 |
| PAPER-028 | Track streak data (winning streaks, losing streaks) | bookie-emulator | P2 |
| PAPER-029 | Enforce bankroll management rules: max bet size, Kelly criterion sizing | bookie-emulator | P2 |

### Bet Ledger

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| PAPER-030 | Provide full historical bet ledger with all placement details, grading results, and P&L | bookie-emulator | P0 |
| PAPER-031 | Filter bet ledger by sport, league, bet type, date range, and outcome | bookie-emulator | P1 |
| PAPER-032 | Sort bet ledger by date, edge size, stake, profit/loss | bookie-emulator | P1 |

---

## 4. Interface Features

Features for the three equal first-class interfaces.

### CLI Features

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| CLI-001 | View current detected +EV edges with filtering by sport, league, bet type, and minimum edge size | cli | P0 |
| CLI-002 | View prediction summaries with calibrated probabilities and confidence intervals | cli | P0 |
| CLI-003 | View simulation result summaries (key percentiles, expected outcomes) | cli | P1 |
| CLI-004 | Place paper bets from the command line | cli, bookie-emulator | P0 |
| CLI-005 | View paper bet history and ledger with filters | cli, bookie-emulator | P0 |
| CLI-006 | View paper trading performance metrics (ROI, win rate, CLV, units) | cli, bookie-emulator | P0 |
| CLI-007 | Look up current lines and odds for specific games | cli, lines-service | P0 |
| CLI-008 | Look up line movement history for specific games | cli, lines-service | P1 |
| CLI-009 | Look up team and player statistics | cli, statistics-service | P1 |
| CLI-010 | Query the LLM analyst with natural language questions (e.g., "Why do you like the over in this game?") | cli, agent | P0 |
| CLI-011 | Trigger on-demand pipeline runs | cli, agent | P1 |
| CLI-012 | Monitor system health and pipeline status | cli, agent | P1 |
| CLI-013 | Format output as ASCII tables and charts for terminal display | cli | P0 |
| CLI-014 | Handle authentication and user configuration locally | cli | P0 |
| CLI-015 | View feature importance breakdown for a specific prediction | cli, prediction-engine | P1 |
| CLI-016 | View today's full slate of games with predictions across all leagues | cli | P0 |

### UI Features

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| UI-001 | Edge dashboard: display all detected +EV edges with filtering by sport, league, bet type, date, and edge size | ui | P0 |
| UI-002 | Edge dashboard: sorting by edge size, expected value, confidence, and game time | ui | P0 |
| UI-003 | Prediction detail view: calibrated probabilities, confidence intervals, and feature importance | ui | P0 |
| UI-004 | Simulation distribution visualizations: histograms, probability curves, percentile breakdowns | ui | P1 |
| UI-005 | Line movement charts: visual time series of how lines have moved from open to current | ui | P1 |
| UI-006 | Paper trading performance dashboard: ROI over time chart | ui | P0 |
| UI-007 | Paper trading performance dashboard: win rate, CLV charts, calibration plots | ui | P1 |
| UI-008 | Paper trading bet ledger view with filters and sorting | ui | P0 |
| UI-009 | Place paper bets through the UI | ui, bookie-emulator | P0 |
| UI-010 | Data exploration tools: filtering by sport, league, date, bet type, edge size | ui | P0 |
| UI-011 | System status dashboard: recent pipeline runs, service health | ui | P1 |
| UI-012 | Interactive LLM analyst chat: ask questions about bets, matchups, and system output | ui, agent | P0 |
| UI-013 | Team and player stats display with profiles | ui, statistics-service | P1 |
| UI-014 | Current lines and odds display across sportsbooks | ui, lines-service | P0 |
| UI-015 | Today's games overview with predictions and edges across all leagues | ui | P0 |
| UI-016 | Responsive web layout (not native mobile, but responsive) | ui | P1 |
| UI-017 | Performance breakdown charts by sport/league and bet type | ui | P1 |
| UI-018 | Browser-local user preferences (favorite leagues, default filters) | ui | P2 |

### MCP Features

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| MCP-001 | Tool: get current edges with filters (sport, league, bet type, min edge size) | mcp-server | P0 |
| MCP-002 | Tool: get prediction details for a specific game or bet | mcp-server | P0 |
| MCP-003 | Tool: place a paper bet | mcp-server, bookie-emulator | P0 |
| MCP-004 | Tool: get paper trading performance summary | mcp-server, bookie-emulator | P0 |
| MCP-005 | Tool: get paper bet history with filters | mcp-server, bookie-emulator | P0 |
| MCP-006 | Tool: look up current lines and odds for a game | mcp-server, lines-service | P0 |
| MCP-007 | Tool: look up line movement for a game | mcp-server, lines-service | P1 |
| MCP-008 | Tool: look up team or player statistics | mcp-server, statistics-service | P1 |
| MCP-009 | Tool: ask the LLM analyst a question about a bet, matchup, or prediction | mcp-server, agent | P0 |
| MCP-010 | Tool: trigger a pipeline run | mcp-server, agent | P1 |
| MCP-011 | Tool: get pipeline and system status | mcp-server, agent | P1 |
| MCP-012 | Tool: get simulation results for a specific matchup | mcp-server, simulation-engine | P1 |
| MCP-013 | Resource: current edges summary (contextual data for LLM) | mcp-server | P1 |
| MCP-014 | Resource: recent paper trading performance summary | mcp-server | P1 |
| MCP-015 | MCP protocol lifecycle: initialization, capability negotiation, tool listing | mcp-server | P0 |
| MCP-016 | Support both stdio and SSE transport modes | mcp-server | P0 |
| MCP-017 | Format responses for LLM consumption (structured but readable) | mcp-server | P0 |
| MCP-018 | Tool: get today's slate of games with predictions | mcp-server | P0 |

---

## 5. Agent Features

Features for agent orchestration/automation and LLM-powered analysis.

### Orchestration and Automation

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| AGT-001 | Manage the full prediction pipeline schedule (when to pull data, simulate, predict, scan for edges) | agent | P0 |
| AGT-002 | Coordinate service calls in correct dependency order: statistics-service -> simulation-engine -> prediction-engine -> lines-service -> bookie-emulator | agent | P0 |
| AGT-003 | Automatically trigger paper bets on edges exceeding a configurable threshold | agent, bookie-emulator | P1 |
| AGT-004 | Manage alerting when new +EV edges are detected (push to CLI, UI, MCP clients) | agent | P0 |
| AGT-005 | Support configurable pipeline schedules (e.g., run 2 hours before first game of the day) | agent | P1 |
| AGT-006 | Handle pipeline failures gracefully with retry logic and error reporting | agent | P1 |
| AGT-007 | Support league-specific pipeline scheduling (e.g., daily for NBA, weekly for NFL) | agent | P2 |

### LLM-Powered Analysis

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| AGT-010 | Generate natural language analysis of detected edges explaining why the edge exists | agent | P0 |
| AGT-011 | Generate natural language analysis of predictions (what factors are driving the prediction) | agent | P0 |
| AGT-012 | Answer user questions about specific bets, matchups, and system output | agent | P0 |
| AGT-013 | Interpret paper trading performance and provide commentary on trends | agent | P1 |
| AGT-014 | Generate game previews with key matchup factors and betting angles | agent | P1 |
| AGT-015 | Provide situational analysis (e.g., "Why did the line move?", "What is the injury impact?") | agent | P1 |
| AGT-016 | Anthropic API integration for all LLM capabilities | agent | P0 |
| AGT-017 | Prompt engineering and management for analysis quality | agent | P0 |
| AGT-018 | Generate daily summary of all edges and recommended bets across leagues | agent | P1 |

---

## 6. Operational Features

Monitoring, alerting, CI/CD, infrastructure management, and observability.

### Infrastructure

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| OPS-001 | Dockerfiles for all services | infra-ops | P0 |
| OPS-002 | docker-compose configuration for local development of the full system | infra-ops | P0 |
| OPS-003 | CI/CD pipeline definitions (GitHub Actions workflows) for all repos | infra-ops | P0 |
| OPS-004 | Deployment configurations for staging environment | infra-ops | P1 |
| OPS-005 | Deployment configurations for production environment | infra-ops | P1 |
| OPS-006 | Service dependency graph for deployment ordering | infra-ops | P1 |
| OPS-007 | Database migration tooling and orchestration | infra-ops | P1 |
| OPS-008 | Secrets management configuration (vault/secret store setup) | infra-ops | P0 |

### Monitoring and Observability

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| OPS-010 | Monitoring infrastructure configuration (Prometheus, Grafana, or equivalent) | infra-ops | P1 |
| OPS-011 | Service health check endpoints for all services | all services | P1 |
| OPS-012 | Centralized logging infrastructure | infra-ops | P1 |
| OPS-013 | Alerting rules for service failures and pipeline errors | infra-ops | P1 |
| OPS-014 | Dashboard for external API health (sports data APIs, odds APIs) | infra-ops | P2 |
| OPS-015 | Pipeline run monitoring: duration, success/failure rates, data freshness | infra-ops, agent | P1 |
| OPS-016 | Model performance monitoring: track prediction calibration and accuracy degradation over time | prediction-engine, infra-ops | P1 |

### GitHub and Development Operations

| ID | Feature | Owner(s) | Priority |
|---|---|---|---|
| OPS-020 | Shared GitHub organization settings: issue templates, PR templates | infra-ops | P0 |
| OPS-021 | Branch protection rules and CODEOWNERS | infra-ops | P0 |
| OPS-022 | Renovate configuration for dependency updates | infra-ops | P1 |
| OPS-023 | Automated test execution in CI for all services | infra-ops | P0 |
| OPS-024 | Environment-specific configuration management (dev, staging, prod) | infra-ops | P1 |

---

## Summary

| Category | P0 | P1 | P2 | Total |
|---|---|---|---|---|
| Core Pipeline | 18 | 11 | 2 | 31 |
| Bet Types | 12 | 15 | 21 | 48 |
| Paper Trading | 7 | 7 | 3 | 17 |
| CLI | 9 | 6 | 0 | 15 |
| UI | 7 | 8 | 1 | 16 |
| MCP | 8 | 6 | 0 | 14 |
| Agent | 7 | 8 | 1 | 16 |
| Operations | 5 | 10 | 1 | 16 |
| **Total** | **73** | **71** | **29** | **173** |
