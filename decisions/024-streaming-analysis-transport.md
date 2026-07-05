# ADR-024: Streaming Analysis Transport

## Status

Accepted (2026-07-05, Phase 5)

## Context

The Phase 5 dashboard's chat sidebar wants token streaming from the agent's LLM analysis so responses feel
live (the roadmap's "streaming response display"). Phase 4 shipped `POST /api/v1/agent/analysis` as strictly
synchronous JSON. Options considered:

1. **Content negotiation on the existing route** (`Accept: text/event-stream`): breaks generated clients —
   oapi-codegen (CLI) and openapi-typescript (UI) model one media type per operation/status, so adding an
   alternate 200 body changes existing generated signatures; FastAPI also cannot cleanly express Accept-based
   negotiation in its exported spec.
2. **WebSocket**: bidirectional transport for a unidirectional stream; heavier proxying, no generated-spec
   representation at all.
3. **A separate SSE route** (`POST /analysis/stream`): purely additive to the spec, trivially representable
   as `200: text/event-stream`, and matches the SSE bridging the UI already does for live updates.

## Decision

Add **`POST /api/v1/agent/analysis/stream`** returning `text/event-stream`, sharing the `AnalysisRequest`
body and all semantics with `POST /analysis`. The provider protocol (ADR-011) gains a `stream()` method on
both Anthropic (SDK `messages.stream`) and Ollama (NDJSON `/api/chat` with `stream: true`); the terminal
chunk carries the same usage-bearing result as `complete()`, so token accounting is identical.

Event grammar (documented in [agent-api.md](../api-contracts/agent-api.md)):

- `chunk` → `{"text": delta}` repeated;
- `done` → the persisted analysis envelope with `meta.cached` (a cache hit replays the stored content as a
  single chunk, no LLM call);
- `error` → `{"code", "message"}` on mid-stream provider failure.

Two invariants:

- **Pre-stream failures stay JSON**: the service awaits the first provider chunk before the response is
  constructed, so validation/unknown-id/degraded-LLM errors return the standard error envelope (422/404/502)
  and the response never switches content type.
- **No partial persistence**: the `agent.analyses` row and Redis cache entry are written only after the
  stream completes; mid-stream failure or client disconnect persists nothing.

## Consequences

### Positive

- Additive spec change: existing CLI/UI generated clients recompile unchanged
- The UI proxies the stream byte-for-byte (`/api/chat`) and falls back transparently when it receives JSON
- Cached analyses replay instantly without token spend

### Negative

- Two routes share analysis semantics — mitigated by the shared `_prepare()`/`_persist()` service internals
- OpenAPI cannot formally type SSE events; the grammar lives in prose in the contract doc

### Neutral

- A cached replay arrives as one large chunk rather than "typing" — consumers may animate if desired
