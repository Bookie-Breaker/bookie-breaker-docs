# Playbook 01 — Installation

From a fresh machine to a running, seeded stack with the `bb` CLI on your PATH. One-time setup; for
day-to-day start/stop see [02 — Daily Operations](02-daily-operations.md).

---

## 1. Prerequisites

Required on PATH before you start:

| Tool                                                  | Why                                                          |
| ----------------------------------------------------- | ------------------------------------------------------------ |
| `git` + SSH access to the `Bookie-Breaker` GitHub org | cloning all 11 repos over `git@github.com:`                  |
| [mise](https://mise.jdx.dev)                          | per-repo toolchain pinning (go, uv, pnpm, lefthook, linters) |
| [task](https://taskfile.dev)                          | the workspace orchestrator                                   |
| Docker + Docker Compose                               | the full stack runs as containers                            |

Optional but recommended: `gh` (CI status in workspace scripts), `graphify` and `flock` (knowledge-graph
automation). Language toolchains are NOT needed globally — mise installs them per repo.

Background: [operations/tool-management.md](../operations/tool-management.md).

---

## 2. Bootstrap the workspace

Clone infra-ops, then run the setup script — it clones the other repos and configures everything:

```bash
mkdir BookieBreaker && cd BookieBreaker
git clone git@github.com:Bookie-Breaker/bookie-breaker-infra-ops.git
./bookie-breaker-infra-ops/workspace/scripts/dev-env-setup.sh
```

The script is idempotent and safe to re-run. It runs preflight checks, clones missing repos, installs mise
toolchains and lefthook hooks per repo, lays the root symlinks (`Taskfile.yml`, `CLAUDE.md`, scripts),
copies `.env.example` → `.env` where missing, and builds the knowledge graphs. Flags: `--dry-run`,
`--skip-clone`, `--skip-graphs`.

Finish by adding the lines it prints to your shell profile:

```bash
eval "$(mise activate zsh)"
export MISE_CONFIG_FILE=".config/mise.toml"
export LEFTHOOK_CONFIG=".config/lefthook.yml"
```

Full detail: [operations/dev-env-setup.md](../operations/dev-env-setup.md).

---

## 3. Configure environment variables

The setup script copied `bookie-breaker-infra-ops/.env.example` to `.env`. Edit
`bookie-breaker-infra-ops/.env` and set what you need:

| Variable                         | Required?               | Notes                                                                       |
| -------------------------------- | ----------------------- | --------------------------------------------------------------------------- |
| `POSTGRES_PASSWORD`              | defaults to `localdev`  | rotates the superuser only — service roles stay `localdev`                  |
| `ODDS_API_KEY`                   | recommended             | [The Odds API](https://the-odds-api.com); free tier is 500 requests/month   |
| `ANTHROPIC_API_KEY`              | optional                | only if you switch the agent's `LLM_PROVIDER` to `anthropic`                |
| `OLLAMA_MODEL`                   | defaults to `phi3:mini` | pulled automatically on first start (~2.2 GB)                               |
| `CFBD_API_KEY`, `CBBD_API_KEY`   | optional                | college football/basketball stats sources                                   |
| `SHARP_API_URL`, `SHARP_API_KEY` | optional                | live in-game odds; see [04 §3](04-parlays-props-and-live.md#3-live-betting) |

Without `ODDS_API_KEY` the stack still works: line ingestion is disabled, but the read APIs serve the
seeded fixture data, so every workflow below can be exercised.

League enablement (`ODDS_API_SPORTS`, `LEAGUES_ENABLED`) ships with sensible defaults — leave them alone
for now and see [06 — Seasonal Operations](06-seasonal-operations.md) when a season starts.

---

## 4. First start

From the workspace root:

```bash
task up          # build + start all containers (first build takes a while)
task db:migrate  # run enum + per-service migrations from the host
task db:seed     # load fixture sportsbooks, sample lines, and paper bets
```

Notes:

- Migrations run from the host, not inside compose — the lines-service, prediction-engine,
  bookie-emulator, and agent repos must be present (the setup script cloned them).
- On first start `ollama-init` pulls the LLM model in the background. Nothing blocks on it; the agent
  serves degraded (LLM features unavailable) until the pull completes.

---

## 5. Install the `bb` CLI

```bash
cd bookie-breaker-cli
task build       # outputs bin/bb
sudo ln -s "$(pwd)/bin/bb" /usr/local/bin/bb   # or add bin/ to PATH
```

Optional shell completion:

```bash
bb completion zsh > "${fpath[1]}/_bb"    # bash/fish variants: bb completion --help
```

Optional config at `~/.config/bookiebreaker/config.yaml` (defaults work for a local stack):

```yaml
default_league: NBA
format: table
timeout: 10s
analysis_timeout: 120s # used by `bb ask`
```

Precedence: defaults < config file < env vars (`AGENT_URL` etc., service URLs only) < flags.

---

## 6. Verify the install

```bash
bb health
```

All five probed services should report healthy. Then check the UIs:

| URL                          | Expect                                                   |
| ---------------------------- | -------------------------------------------------------- |
| <http://localhost:3000>      | dashboard home; `/slate` shows seeded games              |
| <http://localhost:3001>      | Grafana (`admin` / `admin`; anonymous read access is on) |
| <http://localhost:8006/docs> | agent OpenAPI docs                                       |

If something is unhealthy, go to [07 — Troubleshooting](07-troubleshooting.md).

---

## 7. Optional: connect Claude via MCP

The MCP server (port 8007) exposes 15 read/act tools (`get_edges`, `place_bet`, `run_pipeline`, …).
Two ways to connect:

Claude Code, against the running compose stack (HTTP transport):

```bash
claude mcp add --transport http bookiebreaker http://localhost:8007/mcp
```

Claude Desktop, running the server directly over stdio — add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "bookiebreaker": {
      "command": "uv",
      "args": [
        "run",
        "--directory",
        "/path/to/bookie-breaker-mcp-server",
        "python",
        "-m",
        "mcp_server"
      ],
      "env": {
        "AGENT_URL": "http://localhost:8006",
        "LINES_SERVICE_URL": "http://localhost:8001",
        "STATISTICS_SERVICE_URL": "http://localhost:8002",
        "SIMULATION_ENGINE_URL": "http://localhost:8003",
        "PREDICTION_ENGINE_URL": "http://localhost:8004",
        "BOOKIE_EMULATOR_URL": "http://localhost:8005"
      }
    }
  }
}
```

Tool reference: [components/mcp-server.md](../components/mcp-server.md) and the
[mcp-server README](https://github.com/Bookie-Breaker/bookie-breaker-mcp-server#readme).
