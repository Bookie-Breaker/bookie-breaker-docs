# ADR-009: OpenAPI Codegen for Cross-Service Type Sharing

## Status
Accepted

## Context
With 11 repositories spanning 3 languages (Go, Python, TypeScript), we need a strategy for keeping domain models and API client types consistent across services without creating a shared package repo (which adds versioning overhead, publish cycles, and cross-language complexity for a solo developer).

Options considered:
1. **Shared package repo** — A 12th repo published as a pip/Go/npm package. Rejected: versioning overhead is high, cross-language packages are painful, and a solo dev doesn't need the formality.
2. **Accept duplication** — Each service defines its own models. Rejected: model drift is dangerous when the system relies on precise edge calculations.
3. **OpenAPI codegen** — Define API contracts as OpenAPI specs, generate typed API clients and model types per language.
4. **Hybrid** — Shared package for some things, codegen for others. Rejected: unnecessary complexity.

## Decision
Use **OpenAPI code generation** as the sole mechanism for cross-service type sharing.

### How it works:
1. API contracts are defined as OpenAPI 3.1 YAML specs in `bookie-breaker-docs/api-contracts/`
2. Each service's spec defines its request/response schemas (which include domain model types)
3. Code generators produce typed client libraries:
   - **Go clients:** `oapi-codegen` (Echo-compatible server stubs + client code)
   - **Python clients:** `openapi-python-client` or `datamodel-code-generator` (Pydantic models)
   - **TypeScript clients:** `openapi-typescript` (types only, used by UI)
4. Generated code lives in each consuming repo (not committed to the API contracts repo)
5. A Taskfile command regenerates clients when specs change

### What this covers:
- All request/response types between services
- Shared enums (Sport, League, BettingMarketType, GameStatus, etc.)
- API client wrappers with proper typing

### What this does NOT cover:
- Internal-only types (simulation internals, ML feature vectors) — these stay in their owning service
- Event schemas (Redis pub/sub payloads) — defined in `communication-patterns.md`, implemented per-service
- Database schemas — internal to each service

## Consequences

### Positive
- Single source of truth for API contracts (the OpenAPI spec)
- Type-safe API clients in all 3 languages with zero manual synchronization
- Forces API-first design — spec must be written before implementation
- No shared package repo, no versioning, no publish cycles
- OpenAPI specs double as API documentation

### Negative
- Generated code can be verbose or awkward — may need thin wrappers
- Adding a new field requires: update spec → regenerate clients → update consumers
- Code generators have quirks per language that need to be learned
- Event schemas (pub/sub) are not covered by OpenAPI — separate consistency concern

### Neutral
- Spec-first development requires discipline but produces better APIs
- Generated code should not be edited — any customization goes in wrapper layers
