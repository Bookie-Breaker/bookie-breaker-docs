# CI/CD & GitHub Configuration

GitHub Actions pipelines, repository standards, and release strategy for the BookieBreaker organization. All 11
repositories in the [Bookie-Breaker](https://github.com/Bookie-Breaker) GitHub org follow these conventions.

---

## 1. GitHub Actions Strategy

### Reusable Workflows

CI/CD logic lives in reusable workflows in `bookie-breaker-infra-ops/.github/workflows/`. Each service repo calls these
reusable workflows rather than defining its own pipeline from scratch. This keeps CI consistent across the org and
avoids duplicating pipeline logic in 11 repositories.

**Reusable workflow files:**

| Workflow            | File                  | Called By                                                                |
| ------------------- | --------------------- | ------------------------------------------------------------------------ |
| Go CI               | `go-ci.yml`           | lines-service, statistics-service, cli                                   |
| Python CI           | `python-ci.yml`       | simulation-engine, prediction-engine, agent, mcp-server, bookie-emulator |
| SvelteKit CI        | `sveltekit-ci.yml`    | ui                                                                       |
| Docker Build + Push | `docker-build.yml`    | All services with Dockerfiles                                            |
| OpenAPI Codegen     | `openapi-codegen.yml` | bookie-breaker-docs (triggers client regeneration)                       |

**Calling a reusable workflow from a service repo:**

```yaml
# .github/workflows/ci.yml in lines-service
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: Bookie-Breaker/bookie-breaker-infra-ops/.github/workflows/go-ci.yml@main
    secrets: inherit

  docker:
    needs: ci
    if: github.ref == 'refs/heads/main'
    uses: Bookie-Breaker/bookie-breaker-infra-ops/.github/workflows/docker-build.yml@main
    with:
      image-name: lines-service
    secrets: inherit
```

---

## 2. Per-Language CI Pipelines

### Go CI (`go-ci.yml`)

Used by: lines-service, statistics-service, cli

```yaml
name: Go CI (Reusable)

on:
  workflow_call:

jobs:
  go-ci:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v7

      - name: Set up Go
        uses: actions/setup-go@v6
        with:
          go-version-file: go.mod

      - name: Download dependencies
        run: go mod download

      - name: Verify dependencies
        run: go mod verify

      - name: Lint
        uses: golangci/golangci-lint-action@v9
        with:
          version: latest
          args: --config=.config/golangci.yml

      - name: Test
        run: go test -race -coverprofile=coverage.out ./...

      - name: Vulnerability check
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          "$(go env GOPATH)/bin/govulncheck" ./...

      - name: Build
        run: go build -o /dev/null ./cmd/...

      - name: Upload coverage
        uses: codecov/codecov-action@v7
        with:
          files: coverage.out
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: true
```

### Python CI (`python-ci.yml`)

Used by: simulation-engine, prediction-engine, agent, mcp-server, bookie-emulator

```yaml
name: Python CI (Reusable)

on:
  workflow_call:

jobs:
  python-ci:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v7

      - name: Install uv
        uses: astral-sh/setup-uv@v8.2.0

      - name: Set up Python
        run: uv python install

      - name: Install dependencies
        run: uv sync --all-extras --dev

      - name: Lint
        run: uv run ruff check .

      - name: Format check
        run: uv run ruff format --check .

      - name: Type check
        run: uv run mypy src/

      - name: Test
        run: uv run pytest --cov=src/ --cov-report=xml

      - name: Dependency audit
        run: uv run pip-audit

      - name: Upload coverage
        uses: codecov/codecov-action@v7
        with:
          files: coverage.xml
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: true
```

### SvelteKit CI (`sveltekit-ci.yml`)

Used by: ui

```yaml
name: SvelteKit CI (Reusable)

on:
  workflow_call:

jobs:
  sveltekit-ci:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v7

      - name: Install pnpm
        uses: pnpm/action-setup@v6

      - name: Set up Node
        uses: actions/setup-node@v6
        with:
          node-version-file: .nvmrc
          cache: pnpm

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Lint
        run: pnpm run lint

      - name: Format check
        run: pnpm run format:check

      - name: Unit tests
        run: pnpm exec vitest run --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v7
        with:
          files: coverage/lcov.info
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: true

      - name: Build
        run: pnpm run build

      - name: E2E tests
        if: hashFiles('playwright.config.ts', 'playwright.config.js') != ''
        run: |
          pnpm exec playwright install --with-deps
          pnpm exec playwright test

      - name: Dependency audit
        run: pnpm audit --audit-level=high
```

### Docker Build + Push (`docker-build.yml`)

Shared by all services. Builds a multi-platform Docker image, scans it with Trivy, and pushes to the container registry.

```yaml
name: Docker Build (Reusable)

on:
  workflow_call:
    inputs:
      image-name:
        required: true
        type: string

jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      security-events: write
    steps:
      - name: Checkout
        uses: actions/checkout@v7

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v4

      - name: Log in to GHCR
        uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v6
        with:
          images: ghcr.io/bookie-breaker/${{ inputs.image-name }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-,format=long
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push
        uses: docker/build-push-action@v7
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@0.36.0
        with:
          image-ref: ghcr.io/bookie-breaker/${{ inputs.image-name }}:sha-${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: HIGH,CRITICAL
          exit-code: "1"

      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v4
        if: always()
        with:
          sarif_file: trivy-results.sarif
```

### Coverage Gating (Codecov)

Coverage is enforced by Codecov commit statuses, not by the test runners ([ADR-034](../decisions/034-coverage-gating-via-codecov.md)).

- Every code repo (three Go, five Python, ui) versions a `codecov.yml` at its root setting `codecov/project` to
  **85% (±0.5%)** and `codecov/patch` to **85% (±5%)**, with per-repo `ignore` lists for generated code,
  migrations, scripts, and test trees.
- Uploads authenticate with the **org-level `CODECOV_TOKEN` Actions secret** (reaching the reusable workflows via
  `secrets: inherit`) and set `fail_ci_if_error: true` — a failed upload fails the build. Historical note: before
  this was introduced, tokenless uploads had been rejected silently (`Token required - not valid tokenless
upload`) on every run.
- `codecov/project` is a required status check on `main` for the nine code repos; `codecov/patch` is advisory.
- Local equivalents: `task coverage` (Go), `task test` with `--cov=src` (Python, branch coverage via
  `[tool.coverage.run]`), `task test:coverage` (ui).
- README badge convention (flat shields.io, one badge per line, space-free alt text to satisfy MD013): CI status
  from `img.shields.io/github/actions/workflow/status/...`, coverage from `img.shields.io/codecov/c/github/...`,
  plus the repo's tech stack as static badges.

---

## 3. PR & Issue Templates

### Issue Templates

Standardized issue templates live in each repo's `.github/ISSUE_TEMPLATE/` directory. Shared template sources are in
`bookie-breaker-infra-ops/.github/` and synced across repos.

**Bug report (`bug_report.yml`):**

```yaml
name: Bug Report
description: Report a bug or unexpected behavior
labels: [bug]
body:
  - type: textarea
    id: description
    attributes:
      label: Description
      description: What happened?
    validations:
      required: true
  - type: textarea
    id: expected
    attributes:
      label: Expected behavior
      description: What did you expect to happen?
  - type: textarea
    id: reproduce
    attributes:
      label: Steps to reproduce
      description: How can this be reproduced?
  - type: textarea
    id: logs
    attributes:
      label: Relevant logs
      description: Paste any relevant log output
      render: shell
  - type: input
    id: service
    attributes:
      label: Service
      description: Which service is affected?
```

**Feature request (`feature_request.yml`):**

```yaml
name: Feature Request
description: Suggest a new feature or enhancement
labels: [feature]
body:
  - type: textarea
    id: problem
    attributes:
      label: Problem
      description: What problem does this solve?
    validations:
      required: true
  - type: textarea
    id: solution
    attributes:
      label: Proposed solution
      description: How should this work?
  - type: textarea
    id: alternatives
    attributes:
      label: Alternatives considered
      description: What other approaches did you consider?
```

### PR Template

**`.github/pull_request_template.md`:**

```markdown
## What

<!-- Brief description of changes -->

## Why

<!-- Motivation and context -->

## How

<!-- Implementation approach, key decisions -->

## Testing

- [ ] Unit tests pass
- [ ] Integration tests pass (if applicable)
- [ ] Manual testing performed

## Checklist

- [ ] Code follows project conventions
- [ ] Self-reviewed the diff
- [ ] No secrets or credentials in the diff
```

---

## 4. Branch Protection

### Main Branch Rules

All repositories enforce branch protection on `main`:

| Rule                      | Setting | Rationale                                                          |
| ------------------------- | ------- | ------------------------------------------------------------------ |
| Require pull request      | Yes     | All changes go through PR, even for solo dev (creates audit trail) |
| Required approvals        | 1       | One approving review plus code-owner review (`* @jsamuelsen11`)    |
| Require CI to pass        | Yes     | The `ci / <lang>-ci` check and `codecov/project` (85%) must pass   |
| Require up-to-date branch | No      | Avoids unnecessary rebases for a solo dev                          |
| Allow force push          | No      | Protect commit history on main                                     |
| Allow deletions           | No      | Prevent accidental branch deletion                                 |

**Branch naming convention:**

```text
feature/<short-description>
fix/<short-description>
chore/<short-description>
docs/<short-description>
```

---

## 5. Renovate

### Shared Configuration

A shared Renovate preset lives in `bookie-breaker-infra-ops` and is extended by all repos, so update behavior is
consistent across the org and there is exactly one file to change.

**Source of truth:** [`bookie-breaker-infra-ops/renovate-config.json`][preset]. It is not reproduced here — a
copy in prose goes stale the moment the preset changes. The rationale behind each choice is recorded in
[ADR-033](../decisions/033-dependency-update-strategy.md); what follows is a summary of the resulting behavior.

The preset extends `config:best-practices`, which brings Docker image and GitHub Action digest pinning, config
migration, dev-dependency pinning, abandonment detection, and weekly lock file maintenance. On top of that:

- **Nothing automerges.** Branch protection requires an approving review plus code-owner review, which Renovate
  cannot satisfy for its own PRs. Merges are driven from each repo's **Dependency Dashboard** issue.
- **`rangeStrategy: "bump"`** so the Python services' `>=` floors actually move. Without it, a satisfied `>=`
  constraint means Renovate proposes nothing and every Python update hides inside lock file maintenance.
- **Releases must age 5 days** (`minimumReleaseAge`) before a PR is opened, with `prCreation: "not-pending"` so
  the queue only ever shows actionable work.
- **Our own reusable workflows are exempt from digest pinning** — `Bookie-Breaker/bookie-breaker-infra-ops` is
  referenced as `@main` on purpose, and pinning it would freeze every service's CI.
- **Grouping** collapses lockstep releases into single PRs: OpenTelemetry (per language), Python and JS tooling,
  Svelte/Vite, Tailwind, mise-pinned tools, GitHub Actions, and Docker images.
- **Language runtimes are grouped by name, not by manager, and never update automatically.** Python, Go, Node
  and pnpm are each pinned in three or four files owned by different managers (Dockerfile, `.config/mise.toml`,
  `go.mod`, `.nvmrc`, `packageManager`). Grouping by manager would move them in separate PRs. They are gated
  behind dashboard approval because a runtime bump such as Python `3.12` → `3.14` reads as a _minor_ update and
  would otherwise open on its own. `@types/node` moves with the Node runtime for the same reason, and
  `requires-python` is grouped with the Python runtime. Ruff's `target-version` is tool config rather than a
  dependency, so it must be updated by hand alongside.
- **Every dependency carries a version floor.** Renovate needs a constraint to have something to bump. The
  Python services' `[dependency-groups]` dev tools were previously bare names and so never updated
  individually — floors were added so `ruff`, `mypy`, `pytest` and friends are visible as PRs.

[preset]: https://github.com/Bookie-Breaker/bookie-breaker-infra-ops/blob/main/renovate-config.json

**Per-repo `.github/renovate.json` (extends the shared preset):**

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>Bookie-Breaker/bookie-breaker-infra-ops//renovate-config.json"]
}
```

### Update Strategy Summary

Nothing automerges; every row below is merged by hand after CI passes.

| Update Type     | Behavior                                               | Schedule      |
| --------------- | ------------------------------------------------------ | ------------- |
| Patch / minor   | Individual PR, or grouped where packages move together | Weekly window |
| Major           | Held on the Dependency Dashboard; tick to open a PR    | On request    |
| Docker / mise   | Grouped per manager                                    | Weekly window |
| GitHub Actions  | Grouped, pinned to digests                             | Weekly window |
| Lock files      | Single `lock-file-maintenance` PR per repo             | Weekly        |
| Security alerts | Individual PR, labeled `security`, no age gate         | Any time      |

The weekly window is before 8am Monday (America/Chicago), capped at 5 concurrent PRs and 2 per hour per repo.

---

## 6. Release Strategy

### Versioning

All services use [Semantic Versioning](https://semver.org/) (semver). Each service is versioned independently since they
are deployed independently.

**Version format:** `v{MAJOR}.{MINOR}.{PATCH}` (e.g., `v1.2.3`)

- **MAJOR:** Breaking API changes (response envelope changes, removed endpoints)
- **MINOR:** New features, new endpoints, backward-compatible changes
- **PATCH:** Bug fixes, performance improvements, dependency updates

### Conventional Commits

All commit messages follow [Conventional Commits](https://www.conventionalcommits.org/) format:

```text
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

**Types:** `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `perf`, `ci`, `build`

**Examples:**

```text
feat(ingestion): add SharpAPI SSE stream support
fix(cache): handle Redis connection timeout gracefully
chore(deps): bump golangci-lint to v1.58
docs(api): update OpenAPI spec for /lines endpoint
```

### Automated Changelog

GitHub Releases are generated automatically from conventional commits. The changelog groups commits by type:

- **Features** (`feat`)
- **Bug Fixes** (`fix`)
- **Performance** (`perf`)
- **Breaking Changes** (commits with `BREAKING CHANGE` footer or `!` suffix)

### Container Image Tags

Every Docker image pushed to GHCR receives multiple tags:

| Tag Pattern        | Example       | Purpose                       |
| ------------------ | ------------- | ----------------------------- |
| `latest`           | `latest`      | Most recent build from `main` |
| `v{semver}`        | `v1.2.3`      | Immutable release tag         |
| `v{major}.{minor}` | `v1.2`        | Rolling minor version tag     |
| `sha-{commit}`     | `sha-a1b2c3d` | Exact commit traceability     |

### Release Process

1. Merge feature/fix PRs to `main` using conventional commit messages
2. When ready to release, create a Git tag: `git tag v1.2.3 && git push --tags`
3. GitHub Actions detects the tag and triggers the release workflow:
   - Builds and pushes Docker images with semver tags
   - Generates a GitHub Release with changelog from conventional commits
   - Attaches any build artifacts (CLI binaries for Go services)

```yaml
# Release trigger in CI
on:
  push:
    tags:
      - "v*"
```

---

## 7. Shared GitHub Configuration

### Standard Labels

All repositories in the Bookie-Breaker org use a consistent set of labels. These are defined in
`bookie-breaker-infra-ops` and synced across repos using a GitHub Actions workflow with
[EndBug/label-sync](https://github.com/EndBug/label-sync).

| Label              | Color     | Description                                 |
| ------------------ | --------- | ------------------------------------------- |
| `bug`              | `#d73a4a` | Something is not working                    |
| `feature`          | `#0075ca` | New feature or enhancement                  |
| `chore`            | `#e4e669` | Maintenance, refactoring, tooling           |
| `breaking`         | `#b60205` | Breaking change                             |
| `dependencies`     | `#0366d6` | Dependency updates                          |
| `security`         | `#ee0701` | Security vulnerability or concern           |
| `documentation`    | `#0075ca` | Documentation updates                       |
| `good first issue` | `#7057ff` | Good for newcomers                          |
| `wontfix`          | `#ffffff` | Will not be addressed                       |
| `duplicate`        | `#cfd3d7` | Duplicate of another issue                  |
| `blocked`          | `#b60205` | Blocked by another issue or external factor |
| `in-progress`      | `#fbca04` | Currently being worked on                   |

### `.editorconfig`

Every repository includes an `.editorconfig` at the root for consistent formatting across editors:

```ini
# .editorconfig
root = true

[*]
indent_style = space
indent_size = 2
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.go]
indent_style = tab

[*.py]
indent_size = 4

[Makefile]
indent_style = tab

[*.md]
trim_trailing_whitespace = false
```

### `CODEOWNERS`

Every repository includes a `CODEOWNERS` file at `.github/CODEOWNERS`:

```text
# Default owner for all files
* @jsamuelsen11
```

This ensures that all PRs automatically request review from the sole owner, providing a notification mechanism even in a
solo-developer workflow.

### License

All repositories use the MIT license. The `LICENSE` file is included at the root of every repo.

---

## 8. OpenAPI Codegen CI

### Spec Change Detection

When OpenAPI specs change in `bookie-breaker-docs`, a GitHub Actions workflow triggers client regeneration in the
consuming service repos via `repository_dispatch`.

**Workflow in `bookie-breaker-docs`:**

```yaml
# .github/workflows/openapi-dispatch.yml
name: OpenAPI Spec Changed

on:
  push:
    branches: [main]
    paths:
      - "api-specs/**"

jobs:
  dispatch:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        repo:
          - lines-service
          - statistics-service
          - simulation-engine
          - prediction-engine
          - agent
          - bookie-emulator
          - cli
          - ui
    steps:
      - name: Trigger client regeneration
        uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.ORG_PAT }}
          repository: Bookie-Breaker/${{ matrix.repo }}
          event-type: openapi-spec-changed
          client-payload: '{"ref": "${{ github.sha }}"}'
```

**Receiver workflow in each service repo:**

```yaml
# .github/workflows/openapi-regen.yml
name: Regenerate OpenAPI Client

on:
  repository_dispatch:
    types: [openapi-spec-changed]

jobs:
  regenerate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Fetch latest specs
        uses: actions/checkout@v4
        with:
          repository: Bookie-Breaker/bookie-breaker-docs
          path: specs
          ref: ${{ github.event.client_payload.ref }}

      - name: Generate client
        run: |
          # Language-specific codegen command
          # Go: oapi-codegen
          # Python: openapi-python-client
          # TypeScript: openapi-typescript
          make generate-client

      - name: Check for changes
        id: diff
        run: |
          git diff --quiet || echo "changed=true" >> "$GITHUB_OUTPUT"

      - name: Create PR
        if: steps.diff.outputs.changed == 'true'
        uses: peter-evans/create-pull-request@v6
        with:
          title: "chore(api): regenerate OpenAPI client"
          body: |
            Auto-generated from spec change in bookie-breaker-docs.
            Spec commit: ${{ github.event.client_payload.ref }}
          branch: chore/openapi-regen
          labels: chore, dependencies
```

### Codegen Tools by Language

| Language   | Tool                    | Output                                  |
| ---------- | ----------------------- | --------------------------------------- |
| Go         | `oapi-codegen`          | Type-safe client and server interfaces  |
| Python     | `openapi-python-client` | Typed httpx client with Pydantic models |
| TypeScript | `openapi-typescript`    | TypeScript types from OpenAPI schemas   |

This pipeline ensures that API contract changes in the central spec repo automatically propagate to all consuming
services, preventing client/server drift.
