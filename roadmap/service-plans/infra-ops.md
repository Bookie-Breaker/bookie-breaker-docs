# PLANNING: infra-ops

## Service
- **Name:** infra-ops
- **Language:** Shell / YAML
- **Framework:** Taskfile, Docker Compose, GitHub Actions

## Implementation Phase
Phase 1 (Infrastructure & Data Foundation)

## Purpose
Operational glue for the BookieBreaker system. Contains Docker Compose configuration, root Taskfile, database init scripts, CI/CD pipeline definitions, seed data fixtures, and shared environment templates. Not a runtime service.

## Ordered Task List

- [ ] Create repo structure: `docker-compose.yml`, `.env.example`, `init-db/`, `scripts/`, `fixtures/`, `.github/workflows/`
- [ ] Write `docker-compose.yml` with Postgres 16 + TimescaleDB, Redis 7, and network configuration
- [ ] Write database init scripts: `init-db/01-create-schemas.sql` (schemas: `lines`, `predictions`, `emulator`), `init-db/02-create-roles.sql`, `init-db/03-create-shared-enums.sql`
- [ ] Write root `.env.example` with all required environment variables documented
- [ ] Create root `Taskfile.yml` at `BookieBreaker/` level with core tasks: `up`, `down`, `build`, `logs`, `test`, `lint`, `gen`, `db:migrate`, `db:seed`, `db:reset`
- [ ] Add per-service dev tasks to Taskfile: `lines-service:dev`, `statistics-service:dev`, etc.
- [ ] Write seed data script (`scripts/seed-data.sh`) and SQL/Python fixtures for development data
- [ ] Create GitHub Actions workflow templates: CI for Go services, CI for Python services, CI for TypeScript UI
- [ ] Add shared workflow for OpenAPI spec validation
- [ ] Create `.github/ISSUE_TEMPLATE/` and `.github/PULL_REQUEST_TEMPLATE.md`
- [ ] Add Renovate configuration for automated dependency updates
- [ ] Document the full local development setup in a README

## Dependencies
- None (this is the first thing built)

## Complexity
**L** -- Many configuration files but no application logic. Docker Compose and Taskfile require careful orchestration of service dependencies.

## Definition of Done
- [ ] `task up` starts Postgres+TimescaleDB, Redis, and all service containers
- [ ] `task down` stops and removes all containers
- [ ] `task test` runs tests across all repos (once repos exist)
- [ ] `task db:seed` populates development database with realistic fixture data
- [ ] Database schemas (`lines`, `predictions`, `emulator`) are created automatically on first `task up`
- [ ] `.env.example` documents every required and optional environment variable
- [ ] GitHub Actions CI passes for at least one service
- [ ] README explains how to clone all repos, set up environment, and start the stack

## Key Documentation
- [Dev Workflow](../bookie-breaker-docs/operations/dev-workflow.md)
- [System Overview](../bookie-breaker-docs/architecture/system-overview.md)
- [Tech Stack Selection (ADR-010)](../bookie-breaker-docs/decisions/010-tech-stack-selection.md)
- [CI/CD & GitHub Integration](../bookie-breaker-docs/operations/ci-cd-github.md)
