# Git Hooks with Lefthook

Pre-commit, commit-msg, and pre-push hooks enforced across all BookieBreaker repositories.

---

## 1. Why Lefthook

[Lefthook](https://github.com/evilmartians/lefthook) is a fast, polyglot git hook manager.

- **Fast:** Runs hooks in parallel by default
- **Polyglot:** Runs any command — not tied to a single language ecosystem
- **No dependencies:** Single binary, installed via mise (see [Tool Management](tool-management.md))
- **Config-as-code:** `.config/lefthook.yml` checked into each repo

---

## 2. Installation and Activation

Lefthook is installed via mise as part of each repo's `.config/mise.toml`:

```bash
# After cloning a repo:
mise install          # Installs lefthook and all other tools
lefthook install      # Activates hooks in .git/hooks/

# Or use the bootstrap task:
task bootstrap
```

Lefthook reads its config from `.config/lefthook.yml` when invoked with:

```bash
export LEFTHOOK_CONFIG=".config/lefthook.yml"
```

Add this to your shell profile, or each repo's `Taskfile.yml` can set it for task-based invocations.

---

## 3. Hook Overview

| Hook | When | Purpose | Speed |
|------|------|---------|-------|
| **pre-commit** | Before each commit | Lint and format staged files, scan for secrets | Fast (< 10s) |
| **commit-msg** | After writing commit message | Validate Conventional Commits format | Instant |
| **pre-push** | Before pushing to remote | Run full test suite, type checking, security audits | Moderate (30s-2min) |

All hooks run in parallel where possible. Pre-commit hooks operate on staged files only for speed.

---

## 4. File Type Coverage

Every file type in the project has a corresponding linter or formatter enforced via hooks. This table shows the complete coverage:

### Language-Specific Linters/Formatters

| File Type | Tool | Check Command | Fix Command | Repos |
|-----------|------|---------------|-------------|-------|
| `*.go` | golangci-lint | `golangci-lint run` | `golangci-lint run --fix` | Go services |
| `*.go` | gofmt | `gofmt -l` (fails if output) | `gofmt -w` | Go services |
| `*.py` | ruff | `uv run ruff check` | `uv run ruff check --fix` | Python services |
| `*.py` | ruff format | `uv run ruff format --check` | `uv run ruff format` | Python services |
| `*.py` | mypy | `uv run mypy src/` | N/A (type errors must be fixed manually) | Python services |
| `*.ts`, `*.svelte` | eslint | `pnpm exec eslint` | `pnpm exec eslint --fix` | UI |
| `*.ts`, `*.svelte`, `*.css` | prettier | `pnpm exec prettier --check` | `pnpm exec prettier --write` | UI |

### Config and Documentation Linters/Formatters

| File Type | Tool | Check Command | Fix Command | Repos |
|-----------|------|---------------|-------------|-------|
| `*.md` | markdownlint-cli2 | `markdownlint-cli2` | `markdownlint-cli2 --fix` | All |
| `*.yml`, `*.yaml` | yamllint | `yamllint -c .config/yamllint.yml` | N/A (fix manually) | All |
| `*.toml` | taplo | `taplo check` | `taplo format` | All |
| `*.json` | prettier / python | `python3 -m json.tool --no-ensure-ascii` | `prettier --write` | All |
| `*.sh` | shellcheck | `shellcheck` | N/A (fix manually) | All |
| `Dockerfile*` | hadolint | `hadolint` | N/A (fix manually) | All |
| `.github/workflows/*.yml` | actionlint | `actionlint` | N/A (fix manually) | All |
| `*.sql` | — | No automated linter (manual review) | — | Infra-ops |

### Security Scanning

| Scope | Tool | Command | Repos |
|-------|------|---------|-------|
| Staged files (secrets) | gitleaks | `gitleaks protect --staged --verbose` | All |
| Go dependencies | govulncheck | `govulncheck ./...` | Go services |
| Python dependencies | pip-audit | `uv run pip-audit` | Python services |
| Node dependencies | pnpm audit | `pnpm audit --audit-level=high` | UI |

---

## 5. Shared Config Files

### `.config/yamllint.yml`

```yaml
extends: default

rules:
  line-length:
    max: 120
    allow-non-breakable-words: true
    allow-non-breakable-inline-mappings: true
  truthy:
    allowed-values: ["true", "false", "on", "off", "yes", "no"]
    check-keys: false
  comments:
    min-spaces-from-content: 1
  document-start: disable
```

### `.config/markdownlint.jsonc`

```jsonc
{
  // Allow long lines (one sentence per line can be long)
  "MD013": false,
  // Allow duplicate headings in different sections
  "MD024": { "siblings_only": true },
  // Allow inline HTML (for badges, images with attributes)
  "MD033": false,
  // Allow bare URLs
  "MD034": false
}
```

### `.config/commitlint.yml`

```yaml
extends:
  - "@commitlint/config-conventional"

rules:
  type-enum:
    - 2
    - always
    - - feat
      - fix
      - chore
      - docs
      - refactor
      - test
      - perf
      - ci
      - build
  subject-case:
    - 2
    - never
    - - upper-case
      - pascal-case
      - start-case
```

### `.config/taplo.toml`

```toml
[formatting]
align_entries = false
array_auto_collapse = true
array_auto_expand = true
array_trailing_comma = true
column_width = 100
indent_string = "  "
reorder_keys = false
```

---

## 6. Per-Repo Lefthook Configurations

### Go Service (.config/lefthook.yml)

```yaml
pre-commit:
  parallel: true
  commands:
    golangci-lint:
      glob: "*.go"
      run: golangci-lint run --config .config/golangci.yml --new-from-rev=HEAD~1
    gofmt-check:
      glob: "*.go"
      run: 'test -z "$(gofmt -l {staged_files})" || (gofmt -l {staged_files} && exit 1)'
    markdownlint:
      glob: "*.md"
      run: markdownlint-cli2 --config .config/markdownlint.jsonc {staged_files}
    yamllint:
      glob: "*.{yml,yaml}"
      run: yamllint -c .config/yamllint.yml {staged_files}
    taplo-check:
      glob: "*.toml"
      run: taplo check --config .config/taplo.toml {staged_files}
    json-check:
      glob: "*.json"
      run: 'python3 -c "import json,sys; [json.load(open(f)) for f in sys.argv[1:]]" {staged_files}'
    shellcheck:
      glob: "*.sh"
      run: shellcheck {staged_files}
    hadolint:
      glob: "Dockerfile*"
      run: hadolint {staged_files}
    actionlint:
      glob: ".github/workflows/*.{yml,yaml}"
      run: actionlint {staged_files}
    gitleaks:
      run: gitleaks protect --staged --verbose

commit-msg:
  commands:
    commitlint:
      run: commitlint --config .config/commitlint.yml --edit {1}

pre-push:
  parallel: true
  commands:
    go-test:
      glob: "*.go"
      run: go test -race ./...
    govulncheck:
      run: govulncheck ./...
```

### Python Service (.config/lefthook.yml)

```yaml
pre-commit:
  parallel: true
  commands:
    ruff-check:
      glob: "*.py"
      run: uv run ruff check {staged_files}
    ruff-format:
      glob: "*.py"
      run: uv run ruff format --check {staged_files}
    markdownlint:
      glob: "*.md"
      run: markdownlint-cli2 --config .config/markdownlint.jsonc {staged_files}
    yamllint:
      glob: "*.{yml,yaml}"
      run: yamllint -c .config/yamllint.yml {staged_files}
    taplo-check:
      glob: "*.toml"
      run: taplo check --config .config/taplo.toml {staged_files}
    json-check:
      glob: "*.json"
      run: 'python3 -c "import json,sys; [json.load(open(f)) for f in sys.argv[1:]]" {staged_files}'
    shellcheck:
      glob: "*.sh"
      run: shellcheck {staged_files}
    hadolint:
      glob: "Dockerfile*"
      run: hadolint {staged_files}
    actionlint:
      glob: ".github/workflows/*.{yml,yaml}"
      run: actionlint {staged_files}
    gitleaks:
      run: gitleaks protect --staged --verbose

commit-msg:
  commands:
    commitlint:
      run: commitlint --config .config/commitlint.yml --edit {1}

pre-push:
  parallel: true
  commands:
    pytest:
      run: uv run pytest tests/unit/ tests/integration/
    mypy:
      run: uv run mypy src/
    pip-audit:
      run: uv run pip-audit
```

### TypeScript UI (.config/lefthook.yml)

```yaml
pre-commit:
  parallel: true
  commands:
    eslint:
      glob: "*.{ts,svelte}"
      run: pnpm exec eslint {staged_files}
    prettier:
      glob: "*.{ts,svelte,css,json}"
      run: pnpm exec prettier --check {staged_files}
    markdownlint:
      glob: "*.md"
      run: markdownlint-cli2 --config .config/markdownlint.jsonc {staged_files}
    yamllint:
      glob: "*.{yml,yaml}"
      run: yamllint -c .config/yamllint.yml {staged_files}
    taplo-check:
      glob: "*.toml"
      run: taplo check --config .config/taplo.toml {staged_files}
    shellcheck:
      glob: "*.sh"
      run: shellcheck {staged_files}
    hadolint:
      glob: "Dockerfile*"
      run: hadolint {staged_files}
    actionlint:
      glob: ".github/workflows/*.{yml,yaml}"
      run: actionlint {staged_files}
    gitleaks:
      run: gitleaks protect --staged --verbose

commit-msg:
  commands:
    commitlint:
      run: commitlint --config .config/commitlint.yml --edit {1}

pre-push:
  parallel: true
  commands:
    vitest:
      run: pnpm exec vitest run tests/unit/ tests/integration/
    svelte-check:
      run: pnpm exec svelte-check
    pnpm-audit:
      run: pnpm audit --audit-level=high
```

### Docs Repo (.config/lefthook.yml)

```yaml
pre-commit:
  parallel: true
  commands:
    markdownlint:
      glob: "*.md"
      run: markdownlint-cli2 --config .config/markdownlint.jsonc {staged_files}
    yamllint:
      glob: "*.{yml,yaml}"
      run: yamllint -c .config/yamllint.yml {staged_files}
    taplo-check:
      glob: "*.toml"
      run: taplo check --config .config/taplo.toml {staged_files}
    json-check:
      glob: "*.json"
      run: 'python3 -c "import json,sys; [json.load(open(f)) for f in sys.argv[1:]]" {staged_files}'
    shellcheck:
      glob: "*.sh"
      run: shellcheck {staged_files}
    gitleaks:
      run: gitleaks protect --staged --verbose

commit-msg:
  commands:
    commitlint:
      run: commitlint --config .config/commitlint.yml --edit {1}

# No pre-push hooks (no test suite in docs repo)
```

### Infra-Ops Repo (.config/lefthook.yml)

```yaml
pre-commit:
  parallel: true
  commands:
    yamllint:
      glob: "*.{yml,yaml}"
      run: yamllint -c .config/yamllint.yml {staged_files}
    shellcheck:
      glob: "*.sh"
      run: shellcheck {staged_files}
    hadolint:
      glob: "Dockerfile*"
      run: hadolint {staged_files}
    actionlint:
      glob: ".github/workflows/*.{yml,yaml}"
      run: actionlint {staged_files}
    markdownlint:
      glob: "*.md"
      run: markdownlint-cli2 --config .config/markdownlint.jsonc {staged_files}
    taplo-check:
      glob: "*.toml"
      run: taplo check --config .config/taplo.toml {staged_files}
    json-check:
      glob: "*.json"
      run: 'python3 -c "import json,sys; [json.load(open(f)) for f in sys.argv[1:]]" {staged_files}'
    gitleaks:
      run: gitleaks protect --staged --verbose

commit-msg:
  commands:
    commitlint:
      run: commitlint --config .config/commitlint.yml --edit {1}

# No pre-push hooks (no test suite in infra-ops repo)
```

---

## 7. Skipping Hooks

In emergencies, hooks can be bypassed:

```bash
git commit --no-verify         # Skip pre-commit and commit-msg hooks
LEFTHOOK=0 git push            # Skip pre-push hooks
```

This is discouraged. CI enforces the same checks, so skipping locally just delays the failure. If a hook is consistently producing false positives, fix the tool config rather than skipping.

---

## 8. Troubleshooting

| Problem | Solution |
|---------|----------|
| Hooks not running | Run `lefthook install` in the repo |
| "command not found" for a tool | Run `mise install` to ensure tools are installed |
| Hooks are slow | Hooks run in `parallel: true` by default. If still slow, check if a specific command (e.g., mypy) is the bottleneck. Consider moving it to pre-push only. |
| False positives from markdownlint | Add rules to `.config/markdownlint.jsonc` to disable the noisy rule |
| gitleaks blocking a non-secret | Add an inline `# gitleaks:allow` comment or update `.gitleaksignore` |
| yamllint line-length errors | The default max is 120 chars. For specific files, use `# yamllint disable-line rule:line-length` |
| mypy errors on staged files | mypy runs on `src/` directory (not individual files) because it needs full import context |

---

## Key Documentation

- [Tool Management](tool-management.md) — Mise installation and tool versions
- [Repo Standards](repo-standards.md) — Required files including `.config/` directory
- [CI/CD & GitHub Integration](ci-cd-github.md) — CI pipelines that mirror hook checks
- [Testing Strategy](testing-strategy.md) — Test framework details (pre-push runs these)
- [Security Model](security-model.md) — gitleaks and dependency audit rationale
