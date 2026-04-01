# Repository Standards

How every BookieBreaker repository is structured, named, and maintained.

---

## 1. Required Files

Every repository must contain the following files. Phase 0 scaffolds these across all 11 repos before feature work begins.

### Universal (all repos)

| File | Purpose |
|------|---------|
| `.config/mise.toml` | Tool version pinning (see [Tool Management](tool-management.md)) |
| `.config/lefthook.yml` | Git hook configuration (see [Git Hooks](git-hooks.md)) |
| `.config/commitlint.yml` | Commit message lint rules |
| `.config/markdownlint.jsonc` | Markdown lint rules |
| `.config/yamllint.yml` | YAML lint rules |
| `.editorconfig` | Shared editor formatting rules (see [CI/CD & GitHub](ci-cd-github.md) section 7) |
| `.gitignore` | Language-specific + common ignore patterns |
| `.github/CODEOWNERS` | Auto-assigns PR reviewers |
| `.github/pull_request_template.md` | Standardized PR description |
| `.github/ISSUE_TEMPLATE/bug_report.yml` | Bug report form |
| `.github/ISSUE_TEMPLATE/feature_request.yml` | Feature request form |
| `.github/workflows/ci.yml` | Calls reusable workflow from infra-ops |
| `.env.example` | Documented placeholders for all env vars (no real secrets) |
| `README.md` | Purpose, quickstart, API docs link, architecture decisions |
| `LICENSE` | MIT |
| `Taskfile.yml` | Per-repo tasks: `dev`, `lint`, `test`, `build` |
| `renovate.json` | Extends shared preset from infra-ops |

### Go Services (lines-service, statistics-service, cli)

| File | Purpose |
|------|---------|
| `go.mod` / `go.sum` | Go module definition and dependency lock |
| `.config/golangci.yml` | Linter configuration |
| `.config/air.toml` | Hot reload configuration |

### Python Services (simulation-engine, prediction-engine, agent, mcp-server, bookie-emulator)

| File | Purpose |
|------|---------|
| `pyproject.toml` | Project metadata, dependencies, and tool config (`[tool.ruff]`, `[tool.mypy]`) |
| `uv.lock` | Dependency lock file |

### TypeScript (ui)

| File | Purpose |
|------|---------|
| `package.json` / `pnpm-lock.yaml` | Package metadata and dependency lock |
| `eslint.config.js` | Linter configuration |
| `.prettierrc` | Formatter configuration |
| `svelte.config.js` | SvelteKit configuration |
| `tsconfig.json` | TypeScript configuration |

---

## 2. The `.config/` Directory

Non-language-specific tooling configuration lives in `.config/` at each repo root. This keeps the repo root clean and groups all tooling config in one discoverable location.

```
.config/
├── mise.toml              # Tool versions (Go, Python, Node, CLI tools)
├── lefthook.yml           # Git hooks (pre-commit, commit-msg, pre-push)
├── commitlint.yml         # Conventional Commits enforcement
├── markdownlint.jsonc     # Markdown linting rules
├── yamllint.yml           # YAML linting rules
├── taplo.toml             # TOML formatting rules (optional, for repos with TOML files)
└── golangci.yml           # Go linting (Go repos only)
└── air.toml               # Hot reload (Go repos only)
```

Tools are configured to find their config in `.config/` via CLI flags or environment variables. See [Git Hooks](git-hooks.md) for how lefthook commands reference these paths.

---

## 3. Repository Naming

- All repos prefixed with `bookie-breaker-` (e.g., `bookie-breaker-lines-service`)
- Kebab-case names throughout
- Docker Compose service names match the repo name minus the `bookie-breaker-` prefix (e.g., `lines-service`)
- GitHub repo descriptions follow the pattern: `BookieBreaker — {brief purpose}`

---

## 4. Directory Layout per Language

### Go Services

```
bookie-breaker-{service}/
├── .config/                 # Tooling config (see section 2)
├── cmd/
│   └── server/              # (or cli/ for the CLI repo)
│       └── main.go
├── internal/                # Private application code
│   ├── handler/             # HTTP handlers
│   ├── service/             # Business logic
│   ├── repository/          # Data access
│   └── model/               # Domain types
├── pkg/                     # Public library code (if any)
├── tests/
│   ├── unit/                # Unit tests (also co-located *_test.go files)
│   ├── integration/         # Tests against real Postgres/Redis (testcontainers)
│   └── e2e/                 # End-to-end tests against running services
├── go.mod
├── go.sum
├── Dockerfile
├── Taskfile.yml
└── README.md
```

**Note:** Go convention co-locates unit tests as `*_test.go` files alongside source. The `tests/unit/` directory is for test helpers and fixtures. `tests/integration/` and `tests/e2e/` hold tests that require external dependencies.

### Python Services

```
bookie-breaker-{service}/
├── .config/                 # Tooling config (see section 2)
├── src/
│   └── {package}/           # Main package (e.g., simulation_engine/)
│       ├── __init__.py
│       ├── main.py          # FastAPI app entry point
│       ├── api/             # Route handlers
│       ├── core/            # Business logic
│       ├── models/          # Pydantic models
│       └── db/              # Database access (if applicable)
├── tests/
│   ├── unit/                # Unit tests (no external deps)
│   ├── integration/         # Tests against real Postgres/Redis
│   └── e2e/                 # End-to-end tests against running services
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── Taskfile.yml
└── README.md
```

### TypeScript (UI)

```
bookie-breaker-ui/
├── .config/                 # Tooling config (see section 2)
├── src/
│   ├── routes/              # SvelteKit pages
│   ├── lib/
│   │   ├── components/      # Reusable Svelte components
│   │   ├── api/             # API client layer
│   │   └── stores/          # Svelte stores
│   └── app.html
├── static/
├── tests/
│   ├── unit/                # Vitest component and utility tests
│   ├── integration/         # API client tests against real backends
│   └── e2e/                 # Playwright browser tests
├── package.json
├── pnpm-lock.yaml
├── svelte.config.js
├── tsconfig.json
├── eslint.config.js
├── .prettierrc
├── Dockerfile
├── Taskfile.yml
└── README.md
```

### Infra-Ops

```
bookie-breaker-infra-ops/
├── .config/                 # Tooling config (see section 2)
├── docker-compose.yml
├── init-db/                 # Database init scripts
├── scripts/                 # Operational scripts
├── fixtures/                # Seed data
├── .github/workflows/       # Reusable CI/CD workflows
├── Taskfile.yml
└── README.md
```

### Docs

```
bookie-breaker-docs/
├── .config/                 # Tooling config (see section 2)
├── architecture/
├── components/
├── decisions/
├── operations/
├── roadmap/
├── algorithms/
├── api-contracts/
├── schemas/
├── research/
├── images/
└── README.md
```

---

## 5. Test Structure

Every service (including the UI) maintains three test levels:

| Level | Directory | What It Tests | External Deps | When It Runs |
|-------|-----------|---------------|---------------|--------------|
| Unit | `tests/unit/` | Pure logic, data transformations, individual functions | None | Pre-commit (fast), CI |
| Integration | `tests/integration/` | Service code against real databases/caches (testcontainers) | Postgres, Redis, external APIs (VCR) | CI, pre-push |
| E2E | `tests/e2e/` | Full user flows against running service stack | All upstream services | CI (nightly), manual |

### Per-Language Test Commands

| Language | Unit | Integration | E2E |
|----------|------|-------------|-----|
| Go | `go test ./tests/unit/... ./internal/...` | `go test ./tests/integration/...` | `go test ./tests/e2e/...` |
| Python | `uv run pytest tests/unit/` | `uv run pytest tests/integration/` | `uv run pytest tests/e2e/` |
| TypeScript | `pnpm exec vitest run tests/unit/` | `pnpm exec vitest run tests/integration/` | `pnpm exec playwright test` |

See [Testing Strategy](testing-strategy.md) for framework choices, coverage expectations, and CI integration.

---

## 6. Naming Conventions

### Branches

Follow the pattern `{type}/{short-description}`:

| Prefix | Use |
|--------|-----|
| `feature/` | New features |
| `fix/` | Bug fixes |
| `chore/` | Maintenance, dependencies, config |
| `docs/` | Documentation changes |
| `refactor/` | Code restructuring |

Examples: `feature/nba-adapter`, `fix/line-dedup-race-condition`, `chore/upgrade-go-1.23`

See [CI/CD & GitHub](ci-cd-github.md) section 4 for branch protection rules.

### Files

| Language / Type | Convention | Example |
|-----------------|-----------|---------|
| Go | `snake_case.go` | `line_handler.go` |
| Python | `snake_case.py` | `edge_detection.py` |
| TypeScript | `kebab-case.ts` | `api-client.ts` |
| Svelte | `PascalCase.svelte` | `EdgesTable.svelte` |
| Config | `kebab-case` or dotfiles | `docker-compose.yml` |
| SQL migrations | `NNN-description.sql` | `001-create-line-snapshots.sql` |
| Markdown docs | `kebab-case.md` | `edge-detection.md` |
| Shell scripts | `kebab-case.sh` | `seed-data.sh` |

### Variables and Functions

| Language | Variables | Functions | Types/Classes | Constants |
|----------|-----------|-----------|---------------|-----------|
| Go | `camelCase` | `camelCase` (exported: `PascalCase`) | `PascalCase` | `PascalCase` or `ALL_CAPS` |
| Python | `snake_case` | `snake_case` | `PascalCase` | `UPPER_SNAKE_CASE` |
| TypeScript | `camelCase` | `camelCase` | `PascalCase` | `UPPER_SNAKE_CASE` |

### Commit Messages

Conventional Commits format enforced by commitlint (see [Git Hooks](git-hooks.md)):

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `perf`, `ci`, `build`

See [CI/CD & GitHub](ci-cd-github.md) section 6 for full details.

---

## 7. Code Review Expectations

All changes go through pull requests, even as a solo developer. This creates an audit trail and ensures CI runs before merge.

### Self-Review Checklist

Before requesting review (or self-merging):

- [ ] CI passes (lint, test, build)
- [ ] No secrets in the diff (gitleaks pre-commit catches this)
- [ ] Breaking changes flagged with `BREAKING CHANGE:` in commit footer
- [ ] New environment variables added to `.env.example` with documentation
- [ ] API changes update the OpenAPI spec
- [ ] New dependencies are justified (prefer standard library where feasible)
- [ ] Tests cover the change (unit at minimum; integration for API changes; e2e for user flows)

### Branch Protection

The `main` branch on all repos requires:

- Pull request (no direct push)
- CI status checks passing
- No force push
- No branch deletion

See [CI/CD & GitHub](ci-cd-github.md) section 3 for full configuration.

---

## 8. Documentation Standards

### README Template

Every service repo README follows this structure:

```markdown
# bookie-breaker-{service}

Brief one-sentence purpose.

## Quickstart

How to run locally (standalone and via Docker Compose).

## API

Link to OpenAPI spec or key endpoints summary.

## Architecture Decisions

Links to relevant ADRs in bookie-breaker-docs.

## Environment Variables

Table of all env vars with descriptions and defaults.
```

### ADR Format

Follow the template at [decisions/000-template.md](../decisions/000-template.md). Every significant technical decision gets an ADR.

### In-Code Documentation

| Language | Standard | Scope |
|----------|----------|-------|
| Go | Godoc comments | All exported symbols |
| Python | Google-style docstrings | All public functions and classes |
| TypeScript | JSDoc | All exported functions |

Do not over-document. Self-explanatory code does not need comments. Document the "why", not the "what".

### Markdown Style

- ATX headers (`#` not `===`)
- One sentence per line (better diffs)
- Code blocks with language specifier (` ```go `, ` ```python `, etc.)
- Tables for structured data
- Cross-reference other docs with relative links (`[Dev Workflow](dev-workflow.md)`)
- No trailing whitespace
- Enforced by markdownlint-cli2 (see [Git Hooks](git-hooks.md))

---

## 9. Dependency Management

### Lock Files

Always committed to the repo:

| Language | Lock File |
|----------|-----------|
| Go | `go.sum` |
| Python | `uv.lock` |
| TypeScript | `pnpm-lock.yaml` |

### Update Strategy

[Renovate](https://docs.renovatebot.com/) manages automated dependency updates across all repos. Configuration lives in `bookie-breaker-infra-ops/` as a shared preset.

| Update Type | Strategy |
|-------------|----------|
| Patch versions | Grouped into one PR per repo, auto-merge if CI passes |
| Minor versions | Individual PRs, manual review |
| Major versions | Individual PRs, manual review, may require migration |
| GitHub Actions digests | Grouped, auto-merge |
| Docker base image digests | Grouped, auto-merge |

See [CI/CD & GitHub](ci-cd-github.md) section 5 for Renovate configuration details.

### Adding New Dependencies

Before adding a dependency:

1. Check if the standard library or an existing dependency can solve the problem
2. Evaluate the package: maintenance status, security track record, license compatibility (MIT/Apache-2.0/BSD preferred)
3. Run a security audit after adding (`govulncheck`, `pip-audit`, or `pnpm audit`)
4. Update the lock file and commit it with the dependency addition

See [Security Model](security-model.md) section 6 for the dependency security policy.

---

## Key Documentation

- [Tool Management](tool-management.md) — Mise configuration for consistent tool versions
- [Git Hooks](git-hooks.md) — Lefthook pre-commit, commit-msg, and pre-push hooks
- [CI/CD & GitHub Integration](ci-cd-github.md) — Pipelines, templates, Renovate, shared workflows
- [Dev Workflow](dev-workflow.md) — Local development setup
- [Security Model](security-model.md) — Secrets, auth, access control
- [Testing Strategy](testing-strategy.md) — Testing approach across services
- [Tech Stack Selection (ADR-010)](../decisions/010-tech-stack-selection.md) — Per-language tool choices
