# ADR-031: Live Line Ingestion Transport

## Status

Accepted

## Context

Phase 7 Wave 2 activates the ADR-007 live tier: SharpAPI's SSE stream feeds in-game line updates to
lines-service, which the agent turns into live edges (short-TTL, `is_live=true`). Three transport questions:

1. **How does the stream integrate with the existing polling pipeline?** The Odds API polling scheduler is
   battle-tested (dedup, change detection, publish); live frames must not fork a second persistence path.
2. **What happens when the stream drops?** Live betting is precisely when connectivity matters most.
3. **How is this built and tested without SharpAPI credentials?** Per the Phase 7 scope decision, live ships
   code-complete against a stub, verified on the desktop with real credentials later.

## Decision

**1. One persistence path.** The new `internal/adapter/sharpapi` client parses SSE frames into the same
`model.LineSnapshot` shape the Odds API normalizer produces (`is_live=true`, `source="sharpapi"`, decimal→American
conversion, identical selection semantics including three-way Draw). A `LiveConsumer` goroutine funnels them
through the **same dedup + insert + publish path** the polling ingestion uses. `events:lines.updated` gains one
additive field: `is_live: true` (with `source: "sharpapi"`). Downstream reactions branch on the flag; non-live
consumers are unaffected.

**2. Reconnect with backoff; polling is the standing fallback.** The consumer reconnects on stream failure with
exponential backoff (500 ms → 30 s cap, jittered, reset on a successful frame). The REST polling scheduler never
stops — it is the always-on floor, so a dead stream degrades live betting to poll-latency rather than to nothing
(matching the failure-mode sketch in `operations/error-handling.md`). The consumer starts only when
`SHARP_API_URL` is set; CI and default local runs simply have no live consumer.

**3. A canonical stub is part of the system.** infra-ops ships `sharp-stub` (compose profile `stubs`): a
dependency-free SSE server replaying a World Cup live-frame fixture in a loop with keep-alives and fresh
timestamps. The frame schema — `{event_id, sport_key, commence_time, home_team, away_team, bookmaker,
market{key, outcomes[{name, price, point?}]}, captured_at}` with decimal prices and Odds-API-compatible
market/sport keys — is the **contract**: the adapter is written against it, Go unit/integration tests replay it
via httptest, and the end-to-end live pass (`SHARP_API_URL=http://sharp-stub:8010`) exercises
stream → snapshots → `lines.updated{is_live}` → agent live re-evaluation → UI without credentials. When real
SharpAPI access arrives, only the adapter's frame-mapping layer can differ — and it is confined to one file.

**Downstream (agent):** live updates trigger a debounced (default 5 s per game, trailing-edge, one in-flight)
trimmed re-evaluation — live-state re-simulation, prediction refresh, edge detection on current lines — producing
`is_live=true` edges with a short TTL (default 120 s). Gated by `LIVE_EDGES_ENABLED` (default off), like the
parlay scanner. Live game state is derived coarsely in v1 (score from statistics-service; `fraction_remaining`
estimated from elapsed wall-clock per sport) — refined when a richer in-game state source exists.

## Consequences

### Positive

- No second write path: live snapshots get dedup, prop columns, closing-line capture, and publishing for free
- Sub-second frame bursts cannot melt the pipeline — the agent-side debounce coalesces per game
- The stub-as-contract makes the WC live pass credential-free and CI-testable, and pins exactly what a real
  SharpAPI integration must map from

### Negative

- The stub frame schema is our design, not SharpAPI's documented format — the real integration may need adapter
  rework (bounded to `internal/adapter/sharpapi`)
- Coarse `fraction_remaining` estimation misprices clock-sensitive live edges (documented approximation; the
  short live-edge TTL limits exposure)
- SSE-per-book-per-market frames mean bursty writes during goal events; TimescaleDB ingest and the dedup filter
  absorb this, but live-volume behavior is unverified until the desktop pass

### Neutral

- Live bets settle on final score through the unchanged grading paths (`is_live` is bookkeeping, not settlement
  semantics); live parlays are out of scope for v1
- The `LIVE` market_type enum value remains unused — live-ness is a flag on real markets, per the Wave 0 decision
