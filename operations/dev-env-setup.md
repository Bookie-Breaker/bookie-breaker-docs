# Dev Environment Setup

How the BookieBreaker workspace is bootstrapped and kept current. All workspace-level assets (orchestrator
Taskfile, repository manifest, utility scripts, shared Claude config, knowledge-graph automation) live in
`bookie-breaker-infra-ops/workspace/` and are symlinked into the `BookieBreaker/` root directory.

---

## 1. Bootstrap a Fresh Machine

Clone infra-ops, then run the setup script — it clones the other repos and configures everything:

```bash
mkdir BookieBreaker && cd BookieBreaker
git clone git@github.com:Bookie-Breaker/bookie-breaker-infra-ops.git
./bookie-breaker-infra-ops/workspace/scripts/dev-env-setup.sh
```

The script is idempotent — safe to re-run at any time. Flags: `--dry-run` (print actions without executing),
`--skip-clone`, `--skip-graphs`.

Steps performed, in order:

1. **Preflight** — requires `git`, `mise`, and `task` on PATH; warns about missing optional tools
   (`docker`, `gh`, `graphify`, `flock`). Language toolchains (go, uv, pnpm, lefthook) are NOT required
   globally — mise pins them per repo.
2. **Clone** — clones any repo from `workspace/repos.txt` missing under the workspace root. Existing repos
   are left untouched (use `refresh-all-repos.sh` to pull).
3. **Toolchains** — `mise trust` + `mise install` per repo against `.config/mise.toml`.
4. **Hooks** — `LEFTHOOK_CONFIG=.config/lefthook.yml lefthook install` per repo. This creates the
   pre-commit, commit-msg, pre-push, and post-commit stubs in `.git/hooks/`.
5. **Symlinks** — lays the root symlink map below with `ln -sfn`.
6. **Env files** — copies `.env.example` → `.env` where `.env` does not exist (never overwrites).
7. **Knowledge graphs** — `update-graphify.sh build` (extract every repo + merge; see section 3).

Finish by adding the shell profile lines the script prints:

```bash
eval "$(mise activate zsh)"
export MISE_CONFIG_FILE=".config/mise.toml"
export LEFTHOOK_CONFIG=".config/lefthook.yml"
```

---

## 2. Root Symlink Map

The `BookieBreaker/` root directory is not a git repository; everything in it is either a cloned repo, the
local `graphify-out/` output, or a symlink into `bookie-breaker-infra-ops/workspace/`:

```text
CLAUDE.md                     -> bookie-breaker-infra-ops/workspace/CLAUDE.md
Taskfile.yml                  -> bookie-breaker-infra-ops/workspace/Taskfile.yml
repos.txt                     -> bookie-breaker-infra-ops/workspace/repos.txt
.graphifyignore               -> bookie-breaker-infra-ops/workspace/.graphifyignore
bookie-breaker.code-workspace -> bookie-breaker-infra-ops/workspace/bookie-breaker.code-workspace
dev-env-setup.sh              -> bookie-breaker-infra-ops/workspace/scripts/dev-env-setup.sh
update-graphify.sh            -> bookie-breaker-infra-ops/workspace/scripts/update-graphify.sh
clone-all.sh                  -> bookie-breaker-infra-ops/workspace/scripts/clone-all.sh
checkout-main-all.sh          -> bookie-breaker-infra-ops/workspace/scripts/checkout-main-all.sh
git-status-all.sh             -> bookie-breaker-infra-ops/workspace/scripts/git-status-all.sh
refresh-all-repos.sh          -> bookie-breaker-infra-ops/workspace/scripts/refresh-all-repos.sh
```

`workspace/repos.txt` is the single source of truth for the repo list (columns: name, stack, required CI
check). Scripts read it via `workspace/scripts/lib/common.sh`; the Taskfile reads it with awk. Adding a
repo to the workspace means adding one line there.

Note the shared Claude config `CLAUDE.md` moved here from `bookie-breaker-docs/` so that one infra-ops
clone bootstraps the entire workspace; see [Claude Config](claude-config.md).

---

## 3. Knowledge Graphs (graphify)

Each repo owns a **gitignored** `graphify-out/` directory built with `graphify extract <repo> --code-only`
— pure AST extraction: no LLM, no API key, seconds per repo. The root `graphify-out/graph.json` is the
merge of all per-repo graphs (`graphify merge-graphs`) and is the graph that `graphify query` and the
shared CLAUDE.md point at.

### Automatic updates

Every repo's lefthook config has a `post-commit` job that:

1. Skips entirely in CI (`$CI`/`$GITHUB_ACTIONS`) and during merges/rebases.
2. Touches a dirty marker (`graphify-out/.dirty/<repo>`) at the workspace root.
3. Spawns a detached, flock-guarded runner (`update-graphify.sh run`).

The runner waits until commits stop for `GRAPHIFY_SETTLE_SECONDS` (default 60), then re-extracts each
dirty repo once and performs a single merge — committing to five repos in a burst produces one batch of
work, not five. Runner activity is logged to `graphify-out/update.log`.

### Manual operations

```bash
task graph:build    # full build: extract all repos + merge (initial setup / recovery)
task graph:update   # process the dirty queue right now (no settle wait)
task graph:merge    # merge existing per-repo graphs only
task graph:status   # queue, lock, and log state
```

### Notes and remedies

- **Do not run `graphify update .` at the workspace root** — it would replace the merged graph with a
  monolithic rescan of all repos.
- After deleting lots of code, graphify's shrink-guard may refuse to write a smaller graph; force with
  `GRAPHIFY_FORCE=1 task graph:build`.
- Doc/semantic content is not extracted by the hooks (AST only). Semantic refreshes of docs/papers remain
  a manual `/graphify` session activity.
- Install graphify with the SQL extractor included: `uv tool install "graphifyy[sql]"`. A plain `graphifyy`
  install skips `.sql` files (init-db schemas, migrations) with a warning; after adding the extra, pick the
  skipped files up with `GRAPHIFY_FORCE=1 graphify extract bookie-breaker-infra-ops --code-only` + `task graph:merge`.

---

## 4. Day-to-Day Utilities

| Command                                                                   | Purpose                                                                       |
| ------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| `./git-status-all.sh`                                                     | Branch, sync, working-tree, CI, PR, and issue overview for every repo         |
| `./refresh-all-repos.sh`                                                  | Stash → switch to default branch → pull --ff-only → prune → restore, per repo |
| `./checkout-main-all.sh`                                                  | Switch every clean repo to main (never stashes or pulls)                      |
| `./clone-all.sh`                                                          | Clone missing repos only (subset of dev-env-setup.sh)                         |
| `task test` / `task lint`                                                 | Run every repo's own test/lint task (repos without one are skipped)           |
| `task test:one REPO=x` / `task lint:one REPO=x` / `task hooks:one REPO=x` | Single-repo variants                                                          |
| `task hooks`                                                              | Run lefthook pre-commit + pre-push checks across all repos                    |

---

## 5. VS Code

Open the multi-root workspace from the root:

```bash
code bookie-breaker.code-workspace
```

Each repo commits `.vscode/extensions.json` (stack-appropriate extension recommendations) and
`.vscode/settings.json` (pointing tools at the non-standard `.config/` paths). All other `.vscode/`
contents stay gitignored.

---

## 6. Branch Protection

Every repo's `main` is protected: 1 approving CODEOWNER review + the repo's CI check are required,
force-pushes and deletions are blocked, and conversation resolution is required. `enforce_admins` is off,
so the org admin can bypass the review requirement when merging their own PRs (GitHub does not allow
self-approval). See [Repo Standards](repo-standards.md).
