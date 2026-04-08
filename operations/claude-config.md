# Centralized Claude Configuration

Strategy for shared Claude Code configuration across all BookieBreaker repositories, enabling AI-assisted development with full system context.

---

## Overview

A shared `CLAUDE.md` lives in `bookie-breaker-docs/` and is symlinked from the parent `BookieBreaker/` directory. This provides Claude Code with system-wide context regardless of which repo the developer is working in. Individual service repos extend this with service-specific context.

---

## File Layout

```
BookieBreaker/
├── CLAUDE.md                          → symlink to bookie-breaker-docs/CLAUDE.md
├── bookie-breaker-docs/
│   └── CLAUDE.md                      # Shared system-wide config (source of truth)
├── bookie-breaker-lines-service/
│   └── CLAUDE.md                      # Service-specific config
├── bookie-breaker-statistics-service/
│   └── CLAUDE.md                      # Service-specific config
├── bookie-breaker-simulation-engine/
│   └── CLAUDE.md                      # Service-specific config
└── ... (each service repo has its own CLAUDE.md)
```

## Symlink Setup

```bash
# From BookieBreaker/ root
ln -sf bookie-breaker-docs/CLAUDE.md CLAUDE.md
```

This is included in the `bootstrap` Taskfile task and `clone-all.sh` script (Phase 0).

---

## Shared CLAUDE.md Contents

The shared config should include:

### System Architecture

- High-level description of BookieBreaker (10 services + docs repo)
- Service map with responsibilities and language per service
- Communication patterns (REST + Redis pub/sub)
- Data flow: statistics → simulation → prediction → edge detection → paper trading

### Repository Layout

- All 11 repos with purpose and language
- Mono-directory structure under `BookieBreaker/`
- Where to find docs, schemas, API contracts, ADRs

### Conventions

- API URL pattern: `/api/v1/{service}/{resource}`
- Shared tooling: mise, lefthook, commitlint, Taskfile
- Commit message format (conventional commits)
- PR template and review process
- Testing strategy: unit, integration, e2e per service

### Shared Commands / Workflows

- `task up` — start full stack via Docker Compose
- `task down` — stop all services
- `task build` — build all services
- `task test` — run tests across all repos
- `task logs` — aggregate logs
- `task bootstrap` — install tools and hooks across all repos

### Cross-Service Development Notes

- OpenAPI codegen pipeline for Go ↔ Python client generation
- OTEL instrumentation requirements for all services
- Redis pub/sub event envelope format
- Database schema ownership per service

---

## Service-Specific CLAUDE.md Contents

Each service repo's `CLAUDE.md` extends the shared config with:

- Service purpose and scope (from component docs)
- Language-specific conventions (Go project layout, Python src layout, etc.)
- Key files and entry points
- Service-specific test commands
- Local development setup (env vars, database, etc.)
- Dependencies on other services and how to mock/stub them
- Common debugging patterns for this service

---

## Maintenance

- Shared `CLAUDE.md` is maintained in `bookie-breaker-docs` and updated as architecture evolves
- Service-specific `CLAUDE.md` files are maintained in their respective repos
- Updates to shared config are part of the normal PR review process for bookie-breaker-docs
