# ADR-033: Dependency Update Strategy

## Status

Accepted

## Context

Renovate now runs against all 11 repos via the Mend-hosted GitHub App, installed org-wide. Each repo carries a
four-line `.github/renovate.json` that extends a single shared preset,
`bookie-breaker-infra-ops/renovate-config.json`. Bringing that preset up to a defensible standard surfaced three
constraints that the original configuration did not account for:

1. **Branch protection makes bot automerge impossible.** Every repo's `main` requires one approving review and
   code-owner review (`.github/CODEOWNERS` is `* @jsamuelsen11` everywhere). Renovate cannot approve its own
   pull requests, so the preset's three `automerge: true` rules could never have merged anything — they would
   have produced a growing pile of green-but-stuck PRs.
2. **`>=` constraints defeat the default range strategy.** All five Python services pin dependencies as
   `fastapi>=0.115`, `redis>=5.2`, and so on, with exact versions resolved in `uv.lock`. Renovate's default
   range handling only rewrites a constraint when the new release falls outside it; a `>=` floor is satisfied by
   every subsequent release, so nothing is ever proposed. A local `--dry-run=full` against the agent repo
   confirmed this: sixteen PyPI lookups, zero branches. Every Python update would have been invisible inside a
   single weekly lock-file-maintenance PR.
3. **Eleven repos multiply PR volume.** A configuration that is pleasant on one repo is overwhelming across
   eleven. Ungrouped updates from three ecosystems plus Docker, GitHub Actions, and mise-pinned toolchains
   would arrive faster than a single maintainer can review them.

A fourth consideration is supply-chain risk. The services pull from PyPI, the Go module proxy, and npm, and the
CI workflows execute third-party GitHub Actions. Compromised-release attacks typically get yanked within days of
publication, so the window between publication and adoption is itself a control.

## Decision

**1. No automerge; the Dependency Dashboard is the control surface.** All `automerge` rules are removed rather
than worked around. The alternatives were rejected: relaxing branch protection would weaken review requirements
for human PRs to benefit a bot, and adding a workflow that auto-approves Renovate PRs re-introduces the
rubber-stamp that required review exists to prevent. Renovate opens PRs; a human merges them. Majors are gated
behind `dependencyDashboardApproval` so they appear as dashboard checkboxes and only become PRs when explicitly
requested.

**2. `config:best-practices` is the baseline**, which brings Docker image and GitHub Action digest pinning,
config migration, dev-dependency pinning, abandonment detection, and weekly lock file maintenance. One
exemption: our own reusable workflows deliberately track `@main`, so
`Bookie-Breaker/bookie-breaker-infra-ops` is excluded from digest pinning. Pinning it would freeze every
service's CI at whatever commit was current, converting a deliberately-live reference into a stale one. That
rule matches by exact package name as well as glob, because it is the one rule whose silent failure would be
expensive.

**3. `rangeStrategy: "bump"` globally.** These repos are applications, not published libraries, so there is no
downstream consumer whose resolution we constrain by raising a floor. Bumping makes each update a reviewable,
individually-revertable PR instead of an opaque lock file diff. Python dev dependencies declared in
`[dependency-groups]` are unversioned by design and continue to move only through lock file maintenance.

**4. Releases must age five days before adoption.** `minimumReleaseAge: "5 days"` with
`internalChecksFilter: "strict"` and `prCreation: "not-pending"` means a PR is not even opened until the gate
passes, so the queue reflects work that is actually actionable. Security updates are the explicit exception:
`vulnerabilityAlerts` sets `minimumReleaseAge: null`, `schedule: ["at any time"]`, and bypasses dashboard
approval, because for a known-exploited vulnerability the calculus inverts.

**5. Volume control through scheduling and grouping.** Updates land in a single weekly window (before 8am
Monday, America/Chicago), capped at `prConcurrentLimit: 5` and `prHourlyLimit: 2`. Packages that version in
lockstep are grouped — OpenTelemetry per language, Python and JS tooling, Svelte/Vite, Tailwind, mise-pinned
tools, GitHub Actions, and Docker images — so a coordinated release arrives as one PR rather than a dozen.

**6. One preset, not per-ecosystem presets.** Ecosystem rules are scoped by datasource and coexist in the single
shared file. A Go rule evaluated in a Python repo simply matches nothing, so the cost of carrying all rules
everywhere is zero, while the cost of maintaining separate presets is real drift. The per-repo
`.github/renovate.json` files remain four-line stubs.

## Consequences

### Positive

- Dependency updates are visible per-package rather than buried in lock file diffs, so a regression can be
  traced to a specific bump and reverted on its own.
- Nothing reaches `main` without human review, and branch protection stays uniform for humans and bots.
- The five-day age gate closes the window in which a compromised release is most likely to be live.
- Digest-pinned Actions and base images make CI and builds reproducible and resistant to tag mutation.
- Adding a twelfth repo costs one four-line file.

### Negative

- Every update requires a manual merge; there is no hands-off path for routine patches. With eleven repos this
  is real recurring work, and the weekly window plus grouping is the mitigation rather than a fix.
- Raising `>=` floors means the manifests no longer document the true minimum supported version, only the most
  recently tested one.
- Digest pinning makes diffs less readable — a SHA replaces a legible tag — and depends on Renovate continuing
  to run to stay current.
- Security fixes for a dependency whose update is gated behind dashboard approval as a major still require an
  explicit click.

### Neutral

- Python dev tooling updates flow through lock file maintenance rather than individual PRs, a consequence of
  the unversioned `[dependency-groups]` convention rather than of this decision.
- The preset lives at the repository root of `bookie-breaker-infra-ops` rather than under `.github/`, because
  fourteen references across the workspace point at that path.
- Renovate's own commits bypass `gitfluff`, which runs only as a local lefthook `commit-msg` hook; semantic
  commit titles are produced by Renovate configuration instead.
