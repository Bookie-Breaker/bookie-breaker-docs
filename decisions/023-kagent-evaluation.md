# ADR-023: kagent Evaluation — Defer

## Status

Accepted (2026-07-05, Phase 5)

## Context

The Phase 5 roadmap called for evaluating [kagent](https://kagent.dev/) (CNCF Sandbox, accepted May 2025;
v0.7.x era) as a Kubernetes-native agent orchestration layer with a bundled chat UI and native MCP support —
"adopt if sufficiently mature, otherwise defer to Phase 7." The attraction was replacing or supplementing
the custom SvelteKit chat interface and some agent plumbing with an off-the-shelf framework.

Findings:

- **kagent is Kubernetes-native by design**: agents, model configs, and tools are CRDs deployed via Helm
  onto a cluster. BookieBreaker runs Docker Compose with no Kubernetes control plane
  ([ADR-005](005-containerized-deployment.md) targets Compose for local dev; k8s is a possible future).
  Adopting kagent today means introducing k8s solely to host a chat surface.
- **Its value adds are already covered**: [ADR-011](011-local-llm-strategy.md)'s provider abstraction gives
  config-switchable Anthropic/Ollama; the Phase 4 mcp-server already exposes BookieBreaker as MCP tools to
  any MCP client; [ADR-017](017-llm-chat-interface-placement.md) already committed to an in-context chat
  sidebar whose core value is page-context injection — something a generic chat UI cannot do.
- **Maturity**: CNCF Sandbox, pre-1.0, weekly release cadence — API stability is not yet at the level worth
  re-platforming for.

## Decision

Defer kagent. Phase 5 ships the custom SvelteKit chat sidebar per ADR-017, backed by the agent's streaming
analysis endpoint ([ADR-024](024-streaming-analysis-transport.md)). Re-evaluate in Phase 7 alongside any
orchestration/deployment changes (if a Kubernetes move happens, kagent becomes a candidate for the agent
runtime, not just the chat surface).

## Consequences

### Positive

- No Kubernetes dependency added to a Compose-based project
- The chat interface keeps page-context injection, which is the feature that makes it useful
- No bet on a pre-1.0 Sandbox API surface

### Negative

- We maintain our own chat UI (small: one sidebar component + one proxy route)

### Neutral

- The Phase 4 mcp-server means kagent (or any MCP client) can already drive BookieBreaker tools today,
  independent of this decision
