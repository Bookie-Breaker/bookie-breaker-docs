# ADR-025: UI Server Proxy and Redis-to-SSE Bridge

## Status

Accepted (2026-07-05, Phase 5)

## Context

The dashboard needs to call six backend services and react to Redis pub/sub events from a browser. Two
architectural questions:

1. **How does the browser reach the backends?** The four Python services (agent, bookie-emulator,
   prediction-engine, simulation-engine) expose no CORS middleware; only the two Go services allow `*`.
   Options: add CORS to the Python services and call them directly (browser-exposed `PUBLIC_*` URLs), or
   route everything through the SvelteKit server.
2. **How do Redis events reach the browser?** Browsers cannot subscribe to Redis; something server-side must
   bridge (the roadmap's own risk mitigation suggested an SSE endpoint).

## Decision

**All backend access is server-side.** `+page.server.ts` load functions and an **allowlist** of `/api/*`
proxy routes call services over internal URLs (`$env/dynamic/private`; Docker DNS names in compose). No CORS
changes to any backend; no service URL or port is exposed to the browser. The proxy is deliberately not a
catch-all: each interactive action (place bet, run pipeline, acknowledge alert, chat, cursor load-more, lazy
distribution fetch) has an explicit route with fixed base URL and path, whitelisted query params, stripped
inbound cookies, timeouts, and error-envelope passthrough — keeping the UI's write surface auditable and
closing the SSRF-shaped hole a generic proxy would open.

**Live updates ride one Redis-to-SSE bridge.** The SvelteKit server holds a single shared Redis subscriber
(all `events:*` channels the UI cares about) fanned out to every open `GET /api/events` connection, with
heartbeats and graceful heartbeat-only degradation when `REDIS_URL` is unset. The client maps events to
debounced `invalidate("app:<domain>")` calls — **event payloads are never rendered** (they carry identifiers
only per redis-schemas.md, and this also sidesteps the fraction-vs-points `edge_percentage` mismatch between
events and REST); pages refetch through their load functions. Missed events (pub/sub is fire-and-forget) are
healed by a full invalidation on reconnect.

## Consequences

### Positive

- Zero backend changes for browser access; one origin; secrets and internal hostnames never leave the server
- Server-side fetches participate in OTEL tracing (ui → agent → downstream) via the undici instrumentation
- Refetch-on-event keeps load functions the single source of truth — no client-side cache to reconcile

### Negative

- Every interactive action needs an explicit proxy route (small, and intentionally so)
- One extra hop for reads (server proxy) — negligible on the compose network

### Neutral

- The two Go services' permissive CORS remains but is unused by the UI
- If auth lands later (api-contracts README defers it), the proxy is the single place to attach it
