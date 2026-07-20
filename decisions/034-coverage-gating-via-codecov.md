# ADR-034: Coverage Gating via Codecov

## Status

Accepted

## Context

CI has produced coverage artifacts since the first workflows landed — `go test -coverprofile` in Go repos and
`pytest --cov` in Python repos — and every run uploaded them with `codecov/codecov-action`. Two problems surfaced
when coverage was finally audited:

1. **Every upload had been failing silently.** Codecov stopped accepting anonymous ("tokenless") uploads for this
   workload; each CI run logged `Upload queued for processing failed: {"message":"Token required - not valid
tokenless upload"}` and continued green because the action's default does not fail the job. Codecov therefore
   never received a single report, and no coverage history exists.
2. **Nothing enforced a floor.** No repo had a `codecov.yml`, a `--cov-fail-under`, a vitest threshold, or a
   `go tool cover` check. The UI repo produced no coverage at all — `@vitest/coverage-v8` was not even installed.

The goal is a uniform 85% coverage target across the nine code repos (three Go, five Python, one
SvelteKit/TypeScript), enforced on every pull request, with visible badges.

## Decision

**Enforce coverage through Codecov commit statuses, configured by a versioned `codecov.yml` in each repo.**

- Each code repo carries a `codecov.yml` setting `codecov/project` at **85% (±0.5%)** and `codecov/patch` at
  **85% (±5%)**, with per-repo `ignore` lists for generated code (`internal/client` in cli, `src/lib/api/gen` in
  ui), migrations, scripts, and test trees.
- The reusable workflows authenticate uploads with an org-level `CODECOV_TOKEN` Actions secret (flowing through
  `secrets: inherit`) and set `fail_ci_if_error: true`, so a failed upload is a failed build — silent regression
  of the pipeline itself is no longer possible.
- The SvelteKit workflow runs `vitest run --coverage` (v8 provider, lcov) and uploads like the others.
- Local parity: Python repos define `[tool.coverage.run] branch = true, source = ["src"]` and run
  `pytest --cov=src` from `task test`; Go repos gain a `task coverage`; the UI gains `task test:coverage`.
- `codecov/project` becomes a required status check on `main` in the nine code repos once uploads are verified;
  `codecov/patch` stays advisory.
- READMEs carry flat shields.io badges: CI status, Codecov coverage, and the repo's tech stack.

**Alternatives rejected:**

- **Tool-native gates** (`--cov-fail-under=85`, vitest `coverage.thresholds`, a `go tool cover -func` script):
  three different mechanisms to keep in sync across three toolchains, no PR-diff coverage view, no trend history,
  and Go has no native threshold support at all. Codecov gives one cross-language enforcement point with PR
  annotations.
- **Self-hosted badges** (gist + shields dynamic endpoint, or committed SVGs): avoids the third-party dependency
  but adds a PAT secret, per-repo badge-update steps, and badge churn in git history — more moving parts than the
  service it replaces, without providing enforcement.

## Consequences

- Repos below 85% show a red `codecov/project` status immediately; once the check becomes required, PRs there
  cannot merge unless they raise coverage. This is accepted as deliberate pressure to backfill tests. Measured
  starting points: agent ~68% (unit only), statistics-service ~55%, ui ~33%, cli ~20% (raw figure includes
  generated clients that Codecov ignores).
- Coverage numbers are branch coverage for Python (via `[tool.coverage.run] branch = true`) and statement
  coverage for Go/TS — the 85% bar is per-repo, not comparable across languages.
- The pipeline depends on Codecov availability; `fail_ci_if_error: true` means a Codecov outage blocks merges
  until it recovers or the step is temporarily relaxed.
- The Codecov GitHub App must remain installed on the org, and the `CODECOV_TOKEN` org secret must remain valid;
  rotating it is a single org-level operation.
