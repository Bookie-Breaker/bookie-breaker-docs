# Bookie Breaker

[![CI](https://img.shields.io/github/actions/workflow/status/Bookie-Breaker/bookie-breaker-docs/ci.yml?branch=main&label=CI&logo=githubactions&logoColor=white)](https://github.com/Bookie-Breaker/bookie-breaker-docs/actions/workflows/ci.yml)
![Markdown](https://img.shields.io/badge/Markdown-docs-000000?logo=markdown&logoColor=white)
![OpenAPI](https://img.shields.io/badge/OpenAPI-3.1-6BA539?logo=openapiinitiative&logoColor=white)

![Bookie Breaker Logo](images/logo.png)

A distributed system for sports gambling predictions that identifies +EV (positive expected value) betting edges against
sportsbooks.

**Sports:** NFL, NBA, MLB, NHL, EPL & FIFA World Cup soccer, NCAA Football, Basketball, Baseball, and Hockey
**Approach:** Hybrid prediction — Monte Carlo simulation generates base probabilities, ML models adjust for contextual
factors

---

## Documentation Index

### Architecture

- [System Overview](architecture/system-overview.md) — High-level architecture diagram
- [Data Flow](architecture/data-flow.md) — End-to-end data flow through the system
- [Communication Patterns](architecture/communication-patterns.md) — How services communicate
- [Feature Inventory](architecture/feature-inventory.md) — Complete list of system features
- [Sequence Diagrams](architecture/sequence-diagrams.md) — Detailed interaction sequences

### Components

- [Agent](components/agent.md) — LLM-powered orchestrator and analyst
- [Bookie Emulator](components/bookie-emulator.md) — Paper trading with accuracy tracking
- [CLI](components/cli.md) — Command-line interface
- [Infra Ops](components/infra-ops.md) — Infrastructure, Docker, CI/CD
- [Lines Service](components/lines-service.md) — Betting lines/odds data ingestion and serving
- [MCP Server](components/mcp-server.md) — Model Context Protocol server for LLM integration
- [Prediction Engine](components/prediction-engine.md) — ML models for prediction adjustment
- [Simulation Engine](components/simulation-engine.md) — Monte Carlo game simulations
- [Statistics Service](components/statistics-service.md) — Historical/statistical data processing
- [UI](components/ui.md) — Web dashboard

### API Contracts

- [API Design Principles](api-contracts/README.md)
- [Lines Service API](api-contracts/lines-service-api.md)
- [Statistics Service API](api-contracts/statistics-service-api.md)
- [Simulation Engine API](api-contracts/simulation-engine-api.md)
- [Prediction Engine API](api-contracts/prediction-engine-api.md)
- [Bookie Emulator API](api-contracts/bookie-emulator-api.md)
- [Agent API](api-contracts/agent-api.md)

### Schemas

- [Domain Models](schemas/domain-models.md) — Core domain entities
- [Database Schemas](schemas/database-schemas/) — Per-service storage schemas

### Research

- [Lines Data Sources](research/lines-data-sources.md) — Betting lines API evaluation
- [Statistics Data Sources](research/statistics-data-sources.md) — Sports stats API evaluation
- [Sport Modeling Analysis](research/sport-modeling-analysis.md) — Per-sport simulation considerations

### Algorithms

- [Simulation Algorithms](algorithms/simulation-algorithms.md) — Monte Carlo framework design
- [Prediction Models](algorithms/prediction-models.md) — ML model architecture
- [Edge Detection](algorithms/edge-detection.md) — EV calculation and position sizing

### Operations

- [Repo Standards](operations/repo-standards.md) — Required files, naming conventions, code review expectations
- [Dev Environment Setup](operations/dev-env-setup.md) — Workspace bootstrap, symlink map, knowledge-graph automation
- [Tool Management](operations/tool-management.md) — Mise configuration for consistent tool versions
- [Git Hooks](operations/git-hooks.md) — Lefthook pre-commit, commit-msg, post-commit, and pre-push hooks
- [Dev Workflow](operations/dev-workflow.md) — Local development setup
- [Testing Strategy](operations/testing-strategy.md) — Testing approach across services
- [Monitoring & Observability](operations/monitoring-observability.md) — Logging, metrics, alerting
- [Security Model](operations/security-model.md) — Secrets, auth, access control
- [Error Handling](operations/error-handling.md) — Resilience and failure recovery
- [CI/CD & GitHub Integration](operations/ci-cd-github.md) — Pipelines, templates, Renovate, shared workflows
- [Claude Configuration](operations/claude-config.md) — Shared CLAUDE.md strategy across repos

### Playbooks

Task-oriented operator guides — see the [playbooks index](playbooks/README.md) for a situation → playbook
routing table:

- [01 — Installation](playbooks/01-installation.md) — Fresh machine to running, seeded stack
- [02 — Daily Operations](playbooks/02-daily-operations.md) — Stack lifecycle, ports, health, logs, dashboards
- [03 — Finding and Betting Edges](playbooks/03-finding-and-betting-edges.md) — Slate → edges → paper bets → results
- [04 — Parlays, Props, and Live](playbooks/04-parlays-props-and-live.md) — Phase 7 markets
- [05 — Pipeline and Scheduling](playbooks/05-pipeline-and-scheduling.md) — Manual runs, cron schedules, tuning
- [06 — Seasonal Operations](playbooks/06-seasonal-operations.md) — League enablement at season start/end
- [07 — Troubleshooting](playbooks/07-troubleshooting.md) — Symptom-first triage and recovery
- [08 — Observability](playbooks/08-observability.md) — Grafana, Prometheus, Tempo, and Loki hands-on
- [09 — Maintenance](playbooks/09-maintenance.md) — Updates, graphs, database care, model retraining

### Decisions

- [ADR Template](decisions/000-template.md)
- [001 — Sport-Agnostic Framework](decisions/001-sport-agnostic-framework.md)
- [002 — Hybrid Prediction Approach](decisions/002-hybrid-prediction-approach.md)
- [003 — Paper Trading Mode](decisions/003-paper-trading-mode.md)
- [004 — Three Equal Interfaces](decisions/004-three-equal-interfaces.md)
- [005 — Containerized Deployment](decisions/005-containerized-deployment.md)
- [006 — Best Tool Per Service](decisions/006-best-tool-per-service.md)
- [007 — Lines Data Sources](decisions/007-lines-data-sources.md)
- [008 — Statistics Data Sources](decisions/008-statistics-data-sources.md)
- [009 — Shared Code Strategy (OpenAPI Codegen)](decisions/009-shared-code-strategy.md)
- [010 — Tech Stack Selection](decisions/010-tech-stack-selection.md)
- [011 — Local LLM Strategy](decisions/011-local-llm-strategy.md)
- [012 — OpenTelemetry-First Observability](decisions/012-observability-otel-first.md)
- [013 — Python Postgres Driver](decisions/013-python-postgres-driver.md)
- [014 — Probability Calibration](decisions/014-probability-calibration.md)
- [015 — Pipeline Scheduler](decisions/015-pipeline-scheduler.md)
- [016 — UI API Client Strategy](decisions/016-ui-api-client-strategy.md)
- [017 — UI Chat Interface](decisions/017-ui-chat-interface.md)
- [018 — Football Simulation Granularity](decisions/018-football-simulation-granularity.md)
- [019 — Database Migration Tooling](decisions/019-database-migration-tooling.md)
- [020 — Statistics Data Bridge](decisions/020-statistics-data-bridge.md)
- [021 — OpenAPI Spec Strategy](decisions/021-openapi-spec-strategy.md)
- [022 — MCP SDK and Transport](decisions/022-mcp-sdk-and-transport.md)
- [023 — Kagent Evaluation](decisions/023-kagent-evaluation.md)
- [024 — Streaming Analysis Transport](decisions/024-streaming-analysis-transport.md)
- [025 — UI Server Proxy and SSE Bridge](decisions/025-ui-server-proxy-and-sse-bridge.md)
- [026 — Sport Expansion Scope and Data Sources](decisions/026-sport-expansion-scope-and-data-sources.md)
- [027 — Three-Way Markets and Regulation Settlement](decisions/027-three-way-markets-and-regulation-settlement.md)
- [028 — Parlay Data Model](decisions/028-parlay-data-model.md)
- [029 — Prop Line Representation](decisions/029-prop-line-representation.md)
- [030 — Parlay Joint Probability and Correlated Kelly](decisions/030-parlay-joint-probability-and-correlated-kelly.md)
- [031 — Live Ingestion Transport](decisions/031-live-ingestion-transport.md)
- [032 — Ensemble and Challenger Serving](decisions/032-ensemble-and-challenger-serving.md)
- [033 — Dependency Update Strategy](decisions/033-dependency-update-strategy.md)
- [034 — Coverage Gating via Codecov](decisions/034-coverage-gating-via-codecov.md)

### Roadmap

- [Implementation Phases](roadmap/implementation-phases.md) — 8-phase (0-7) vertical slice build plan (bootstrap, NBA
  first, then expand)
- [Milestones](roadmap/milestones.md) — Key milestones and success criteria

### Per-Repo Planning

Ordered task lists, dependencies, and definition of done per service:

- [infra-ops](roadmap/service-plans/infra-ops.md) — Phase 1: Docker Compose, Taskfile, Postgres, Redis
- [statistics-service](roadmap/service-plans/statistics-service.md) — Phase 1: Sports stats ingestion and caching (Go)
- [lines-service](roadmap/service-plans/lines-service.md) — Phase 1: Betting lines ingestion and serving (Go)
- [simulation-engine](roadmap/service-plans/simulation-engine.md) — Phase 2: Monte Carlo simulations (Python)
- [prediction-engine](roadmap/service-plans/prediction-engine.md) — Phase 2: ML calibration and prediction (Python)
- [agent](roadmap/service-plans/agent.md) — Phase 3/4: Pipeline orchestration and LLM analysis (Python)
- [bookie-emulator](roadmap/service-plans/bookie-emulator.md) — Phase 3: Paper trading system (Python)
- [cli](roadmap/service-plans/cli.md) — Phase 3: Terminal interface (Go/Charm)
- [mcp-server](roadmap/service-plans/mcp-server.md) — Phase 4: MCP tool server (Python)
- [ui](roadmap/service-plans/ui.md) — Phase 5: Web dashboard (SvelteKit)
