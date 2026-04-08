# PLANNING: infra-ops

## Service

- **Name:** infra-ops
- **Language:** Shell / YAML
- **Framework:** Taskfile, Docker Compose, GitHub Actions

## Implementation Phase

Phase 0 (Repository Bootstrap & Developer Tooling) and Phase 1 (Infrastructure & Data Foundation)

## Purpose

Operational glue for the BookieBreaker system. Contains Docker Compose configuration, root Taskfile, database init
scripts, CI/CD pipeline definitions, seed data fixtures, and shared environment templates. Not a runtime service.

## Ordered Task List

**Phase 0 (repo bootstrap):**

- [ ] Create `.config/mise.toml` in each of the 11 repos with language-appropriate tools
- [ ] Create `.config/lefthook.yml` in each repo with language-specific pre-commit, commit-msg, and pre-push hooks
- [ ] Create shared config files in each repo's `.config/`: `commitlint.yml`, `markdownlint.jsonc`, `yamllint.yml`,
      `taplo.toml`
- [ ] Create `.editorconfig` in each repo
- [ ] Scaffold all 11 repos with required files: `.gitignore`, `LICENSE`, `README.md`, `.env.example`, `Taskfile.yml`,
      `renovate.json`
- [ ] Create GitHub templates in each repo: `.github/CODEOWNERS`, `.github/pull_request_template.md`,
      `.github/ISSUE_TEMPLATE/`
- [ ] Create `.github/workflows/ci.yml` in each repo calling reusable workflows from infra-ops
- [ ] Create `tests/unit/`, `tests/integration/`, `tests/e2e/` directories in each service repo
- [ ] Run `mise install && lefthook install` in each repo; verify hooks work
- [ ] Update `clone-all.sh` to include `mise install` and `lefthook install` post-clone
- [ ] Add `bootstrap` task to root `BookieBreaker/Taskfile.yml`

**Phase 1 (infrastructure):**

- [ ] Create repo structure: `docker-compose.yml`, `.env.example`, `init-db/`, `scripts/`, `fixtures/`,
      `.github/workflows/`
- [ ] Write `docker-compose.yml` with Postgres 16 + TimescaleDB, Redis 7, and network configuration
- [ ] Write database init scripts: `init-db/01-create-schemas.sql` (schemas: `lines`, `predictions`, `emulator`),
      `init-db/02-create-roles.sql`, `init-db/03-create-shared-enums.sql`
- [ ] Write root `.env.example` with all required environment variables documented
- [ ] Create root `Taskfile.yml` at `BookieBreaker/` level with core tasks: `up`, `down`, `build`, `logs`, `test`,
      `lint`, `gen`, `db:migrate`, `db:seed`, `db:reset`
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

**L** -- Many configuration files but no application logic. Docker Compose and Taskfile require careful orchestration of
service dependencies.

## Definition of Done

**Phase 0:**

- [ ] `mise install` installs all tools at pinned versions in every repo
- [ ] `lefthook install` succeeds in all 11 repos
- [ ] Conventional Commit enforcement rejects non-conforming messages
- [ ] Gitleaks blocks commits containing secret patterns
- [ ] All file types covered by linters (Go, Python, TS, YAML, JSON, TOML, Markdown, Shell, Dockerfile, GitHub Actions)
- [ ] `task bootstrap` runs mise install + lefthook install for all repos
- [ ] All 11 repos have consistent structure matching `operations/repo-standards.md`

**Phase 1:**

- [ ] `task up` starts Postgres+TimescaleDB, Redis, and all service containers
- [ ] `task down` stops and removes all containers
- [ ] `task test` runs tests across all repos (once repos exist)
- [ ] `task db:seed` populates development database with realistic fixture data
- [ ] Database schemas (`lines`, `predictions`, `emulator`) are created automatically on first `task up`
- [ ] `.env.example` documents every required and optional environment variable
- [ ] GitHub Actions CI passes for at least one service
- [ ] README explains how to clone all repos, set up environment, and start the stack

## Key Documentation

- [Repo Standards](../bookie-breaker-docs/operations/repo-standards.md)
- [Tool Management](../bookie-breaker-docs/operations/tool-management.md)
- [Git Hooks](../bookie-breaker-docs/operations/git-hooks.md)
- [Dev Workflow](../bookie-breaker-docs/operations/dev-workflow.md)
- [System Overview](../bookie-breaker-docs/architecture/system-overview.md)
- [Tech Stack Selection (ADR-010)](../bookie-breaker-docs/decisions/010-tech-stack-selection.md)
- [CI/CD & GitHub Integration](../bookie-breaker-docs/operations/ci-cd-github.md)
