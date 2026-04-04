# Tool Management with Mise

How BookieBreaker ensures consistent tool versions across all development and CI environments.

---

## 1. Why Mise

[Mise](https://mise.jdx.dev/) is a single tool that manages language runtimes (Go, Python, Node.js) and CLI tools (golangci-lint, ruff, pnpm, etc.) across all repos.

**Problems it solves:**

- No more "works on my machine" from version mismatches
- Replaces scattered version managers (nvm, pyenv, goenv)
- Deterministic: `.config/mise.toml` declares exact versions, every environment gets the same
- Automatic activation when entering a directory
- CI-friendly: `mise install` in GitHub Actions

---

## 2. Installation

### macOS

```bash
brew install mise
```

### Linux

```bash
curl https://mise.run | sh
```

### Shell Activation

Add to your shell profile (`.zshrc`, `.bashrc`, etc.):

```bash
eval "$(mise activate zsh)"   # or bash/fish
```

### Verify

```bash
mise --version
```

---

## 3. Configuration

Each repo contains its own `.config/mise.toml` with the full set of tools it needs. This makes every repo self-contained — cloning a single repo and running `mise install` gives you everything required to work on it.

### Configuration Location

Mise looks for config at `.config/mise.toml` when you set the environment variable:

```bash
export MISE_CONFIG_FILE=".config/mise.toml"
```

Add this to your shell profile alongside the mise activation line. Alternatively, mise also supports `.mise.toml` at the repo root — either convention works as long as the repo is consistent.

### Go Service Example (.config/mise.toml)

```toml
[tools]
go = "1.22"
"go:github.com/golangci/golangci-lint/cmd/golangci-lint" = "1.61"
"go:github.com/air-verse/air" = "1.52"
"npm:markdownlint-cli2" = "0.14"
"npm:@commitlint/cli" = "19"
"npm:@commitlint/config-conventional" = "19"
"pipx:yamllint" = "1.35"
"aqua:rhysd/actionlint" = "1.7"
"aqua:evilmartians/lefthook" = "1.7"
"aqua:tamasfe/taplo" = "0.9"
"aqua:zricethezav/gitleaks" = "8.21"
"aqua:koalaman/shellcheck" = "0.10"
"aqua:hadolint/hadolint" = "2.12"
go-task = "3"
```

### Python Service Example (.config/mise.toml)

```toml
[tools]
python = "3.12"
uv = "latest"
"npm:markdownlint-cli2" = "0.14"
"npm:@commitlint/cli" = "19"
"npm:@commitlint/config-conventional" = "19"
"pipx:yamllint" = "1.35"
"aqua:rhysd/actionlint" = "1.7"
"aqua:evilmartians/lefthook" = "1.7"
"aqua:tamasfe/taplo" = "0.9"
"aqua:zricethezav/gitleaks" = "8.21"
"aqua:koalaman/shellcheck" = "0.10"
"aqua:hadolint/hadolint" = "2.12"
go-task = "3"
```

### TypeScript (UI) Example (.config/mise.toml)

```toml
[tools]
node = "22"
"npm:pnpm" = "9"
"npm:markdownlint-cli2" = "0.14"
"npm:@commitlint/cli" = "19"
"npm:@commitlint/config-conventional" = "19"
"pipx:yamllint" = "1.35"
"aqua:rhysd/actionlint" = "1.7"
"aqua:evilmartians/lefthook" = "1.7"
"aqua:tamasfe/taplo" = "0.9"
"aqua:zricethezav/gitleaks" = "8.21"
"aqua:koalaman/shellcheck" = "0.10"
"aqua:hadolint/hadolint" = "2.12"
go-task = "3"
```

### Docs Repo Example (.config/mise.toml)

```toml
[tools]
"npm:markdownlint-cli2" = "0.14"
"npm:@commitlint/cli" = "19"
"npm:@commitlint/config-conventional" = "19"
"pipx:yamllint" = "1.35"
"aqua:evilmartians/lefthook" = "1.7"
"aqua:tamasfe/taplo" = "0.9"
"aqua:zricethezav/gitleaks" = "8.21"
"aqua:koalaman/shellcheck" = "0.10"
"aqua:hadolint/hadolint" = "2.12"
go-task = "3"
```

---

## 4. Tool Version Matrix

Every repo installs the shared tooling set plus its language-specific tools:

### Shared Tools (all repos)

| Tool | Version | Purpose |
|------|---------|---------|
| lefthook | 1.7.x | Git hook runner |
| go-task | 3.x | Task runner (Taskfile.yml) |
| markdownlint-cli2 | 0.14.x | Markdown linter |
| yamllint | 1.35.x | YAML linter |
| taplo | 0.9.x | TOML formatter/linter |
| commitlint | 19.x | Commit message linter |
| gitleaks | 8.21.x | Secret scanner |
| shellcheck | 0.10.x | Shell script linter |
| hadolint | 2.12.x | Dockerfile linter |
| actionlint | 1.7.x | GitHub Actions linter |

### Go Repos (lines-service, statistics-service, cli)

| Tool | Version | Purpose |
|------|---------|---------|
| go | 1.22.x | Go compiler and runtime |
| golangci-lint | 1.61.x | Go linter (aggregates 50+ linters) |
| air | 1.52.x | Hot reload for Go services |

### Python Repos (simulation-engine, prediction-engine, agent, mcp-server, bookie-emulator)

| Tool | Version | Purpose |
|------|---------|---------|
| python | 3.12.x | Python runtime |
| uv | latest | Package manager (fast pip replacement) |

Python linting tools (ruff, mypy) are installed as project dependencies via `uv` rather than mise, since they need access to the project's virtual environment.

### TypeScript Repo (ui)

| Tool | Version | Purpose |
|------|---------|---------|
| node | 22.x (LTS) | Node.js runtime |
| pnpm | 9.x | Package manager |

TypeScript linting tools (eslint, prettier, svelte-check) are installed as project dependencies via `pnpm`.

---

## 5. CI Integration

GitHub Actions workflows install mise to get the same tool versions as local development.

### Using jdx/mise-action

```yaml
# In reusable CI workflows
steps:
  - uses: actions/checkout@v4
  - uses: jdx/mise-action@v2
    with:
      install: true
  - run: mise install
  # Tools are now available at the versions specified in .config/mise.toml
  - run: golangci-lint run
```

The CI reusable workflows (`go-ci.yml`, `python-ci.yml`, `sveltekit-ci.yml`) in infra-ops should use mise-installed tools. This ensures CI and local dev use identical versions.

See [CI/CD & GitHub Integration](ci-cd-github.md) for workflow details.

---

## 6. Workflows

### New Developer Setup

```bash
# 1. Install mise
curl https://mise.run | sh
echo 'eval "$(mise activate zsh)"' >> ~/.zshrc
echo 'export MISE_CONFIG_FILE=".config/mise.toml"' >> ~/.zshrc
source ~/.zshrc

# 2. Clone all repos
cd BookieBreaker
./clone-all.sh

# 3. Install tools for a specific repo
cd bookie-breaker-lines-service
mise install        # Installs Go, golangci-lint, air, lefthook, etc.
lefthook install    # Activates git hooks

# 4. Start working
task dev            # Hot reload development
```

### Adding a New Tool

1. Add the tool to `.config/mise.toml` in every repo that needs it
2. Run `mise install` to verify it installs correctly
3. If it's a linter/formatter, add a lefthook hook in `.config/lefthook.yml` (see [Git Hooks](git-hooks.md))
4. Update this document's tool version matrix
5. Commit the `.config/mise.toml` changes

### Upgrading a Tool Version

1. Update the version in `.config/mise.toml` for each repo
2. Run `mise install` to get the new version
3. Run `task lint` and `task test` to verify compatibility
4. Commit the `.config/mise.toml` changes
5. CI picks up the new version via `mise install`

### Bootstrap Task

Each repo's `Taskfile.yml` should include a `bootstrap` task:

```yaml
tasks:
  bootstrap:
    desc: Install all tools and activate git hooks
    cmds:
      - mise install
      - lefthook install
```

The root `BookieBreaker/Taskfile.yml` runs bootstrap across all repos:

```yaml
tasks:
  bootstrap:
    desc: Install tools and hooks for all repos
    cmds:
      - |
        for repo in bookie-breaker-*/; do
          echo "Bootstrapping $repo..."
          cd "$repo" && task bootstrap && cd ..
        done
```

---

## Key Documentation

- [Repo Standards](repo-standards.md) — Required files and directory layout
- [Git Hooks](git-hooks.md) — Lefthook configuration using mise-installed tools
- [CI/CD & GitHub Integration](ci-cd-github.md) — CI pipelines that mirror local tooling
- [Dev Workflow](dev-workflow.md) — Local development setup
- [Tech Stack Selection (ADR-010)](../decisions/010-tech-stack-selection.md) — Language and framework choices
