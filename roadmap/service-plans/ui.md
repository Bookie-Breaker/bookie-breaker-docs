# PLANNING: ui

## Service
- **Name:** ui
- **Language:** TypeScript 5+
- **Framework:** SvelteKit + Skeleton UI + Apache ECharts

## Implementation Phase
Phase 5 (Dashboard)

## Purpose
Web-based dashboard for visualizing edges, line movement, simulation distributions, paper trading performance, and interacting with the LLM analyst. Provides the richest visual experience of the three interfaces.

## Ordered Task List

- [ ] Initialize SvelteKit project with pnpm, Skeleton UI component library, and Tailwind CSS
- [ ] Set up project structure: `src/routes/`, `src/lib/components/`, `src/lib/api/`, `src/lib/stores/`
- [ ] Implement API client layer: either generated from OpenAPI specs (`pnpm gen:api`) or hand-written fetch wrappers for agent, lines-service, statistics-service, bookie-emulator
- [ ] Set up ECharts integration with svelte-echarts wrapper
- [ ] Build layout: navigation sidebar, header, responsive grid
- [ ] Build **Edges Dashboard** page (`/edges`):
  - [ ] Table of current edges with columns: game, bet type, side, edge size, predicted prob, implied prob, confidence, game time
  - [ ] Filters: sport/league, bet type, minimum edge size
  - [ ] Sorting: by edge size, EV, confidence, game time
  - [ ] Click row to navigate to prediction detail
- [ ] Build **Prediction Detail** page (`/predictions/[id]`):
  - [ ] Calibrated probabilities for spread, total, moneyline
  - [ ] Confidence intervals displayed visually
  - [ ] Feature importance bar chart (ECharts)
  - [ ] Link to place paper bet
- [ ] Build **Today's Slate** page (`/slate`):
  - [ ] All games grouped by league with predictions and edges
  - [ ] Quick-bet buttons
- [ ] Build **Lines** page (`/lines`):
  - [ ] Current odds across sportsbooks for each game
  - [ ] Line movement chart (ECharts time-series): open line to current, one line per sportsbook
- [ ] Build **Simulation Distributions** component:
  - [ ] Score distribution histograms (ECharts)
  - [ ] Margin distribution with spread line overlay
  - [ ] Total distribution with total line overlay
- [ ] Build **Paper Trading Performance** page (`/performance`):
  - [ ] ROI over time line chart (ECharts)
  - [ ] Win rate over time
  - [ ] CLV over time
  - [ ] Cumulative units chart
  - [ ] Breakdown by sport and bet type (bar charts)
  - [ ] Calibration plot: predicted probability vs actual outcome frequency (scatter + diagonal)
- [ ] Build **Bet Ledger** page (`/bets`):
  - [ ] Filterable, sortable table of all paper bets
  - [ ] Columns: date, game, bet type, side, odds, stake, status, P&L, CLV
  - [ ] Filters: sport, bet type, date range, outcome
- [ ] Build **Bet Placement** form component:
  - [ ] Select game, bet type, side, stake
  - [ ] Show current odds and predicted edge
  - [ ] Confirm and place via bookie-emulator API
- [ ] Build **LLM Chat** interface (`/chat` or sidebar panel):
  - [ ] Text input for questions
  - [ ] Streaming response display (SSE from agent)
  - [ ] Context-aware: can reference current game or edge
- [ ] Implement **live updates**:
  - [ ] SvelteKit server endpoint that bridges Redis pub/sub to SSE for the browser
  - [ ] Subscribe to `edge.detected`, `prediction.completed`, bet grading events
  - [ ] Update UI reactively when new data arrives
- [ ] Add responsive layout for desktop and tablet viewports
- [ ] Write Playwright end-to-end tests: edge viewing, bet placement, performance page
- [ ] Write Vitest unit tests for API client and data transformation utilities
- [ ] Create Dockerfile (Node.js) and integrate into Docker Compose
- [ ] Add `.env.example` with `PUBLIC_*` environment variables

## Dependencies
- **agent** (Phase 3/4) for edges, slate, analysis
- **lines-service** (Phase 1) for lines and line movement data
- **statistics-service** (Phase 1) for stats display
- **bookie-emulator** (Phase 3) for bet placement and performance
- **agent LLM integration** (Phase 4) for the chat interface
- **Redis** for live update bridging

## Complexity
**L** -- Many pages and chart components, but each is a relatively standard SvelteKit page consuming REST APIs. ECharts configuration for line movement and distribution charts requires care. Live updates via SSE add moderate complexity.

## Definition of Done
- [ ] Dashboard loads at `localhost:3000` with navigation between all pages
- [ ] Edges table shows current edges with working filters and sorting
- [ ] Prediction detail page shows probabilities, confidence intervals, and feature importance chart
- [ ] Line movement chart renders correctly for any selected game
- [ ] Simulation distribution histograms display
- [ ] Performance page shows ROI, win rate, CLV charts over time
- [ ] Bet ledger displays all paper bets with filters and sorting
- [ ] User can place a paper bet through the UI form
- [ ] LLM chat sends questions and displays responses
- [ ] Live updates work: new edge appears without page refresh
- [ ] Responsive on desktop and tablet
- [ ] Playwright tests pass for core user flows

## Key Documentation
- [UI Component](../bookie-breaker-docs/components/ui.md)
- [Feature Inventory: UI-001 through UI-018](../bookie-breaker-docs/architecture/feature-inventory.md)
- [Tech Stack Selection: SvelteKit + ECharts](../bookie-breaker-docs/decisions/010-tech-stack-selection.md)
