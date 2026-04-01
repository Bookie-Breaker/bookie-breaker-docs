# ADR-016: UI API Client Strategy

## Status
Accepted

## Context
The SvelteKit UI consumes REST APIs from 4 services: agent, lines-service, statistics-service, and bookie-emulator. We need to decide how the API client layer is built.

Options considered:

1. **OpenAPI codegen (openapi-typescript):** Generate TypeScript types from service OpenAPI specs. Provides type safety and automatic sync with backend changes. Requires working codegen pipeline.
2. **Hand-written fetch wrappers:** Manually write TypeScript API clients with hand-maintained types. Full control, no codegen tooling, but types can drift from the backend.
3. **Hybrid:** Generate types only (via openapi-typescript), write fetch functions by hand using those types.

## Decision
Use the **hybrid approach:** generate TypeScript types from OpenAPI specs with `openapi-typescript`, then write thin fetch wrappers by hand using those generated types.

This combines the benefits of both:
- **Type safety from codegen:** When a backend API changes its response shape, the generated types update and TypeScript catches any UI code that doesn't match.
- **Control from hand-written clients:** Fetch wrappers can handle caching, error mapping, loading states, and SvelteKit-specific patterns (server-side fetch, `+page.ts` load functions) without fighting a generated client library.

The codegen step runs via `pnpm gen:api` (or `task gen`) and produces a `src/lib/api/types.ts` file. Fetch wrappers in `src/lib/api/` import these types.

## Consequences

### Positive
- Type safety: backend API changes are caught at compile time in the UI
- No runtime codegen dependency — generated types are committed to the repo
- Fetch wrappers are simple, readable, and can use SvelteKit conventions (load functions, server-side fetch)
- Avoids the complexity and opinions of a full generated client library (e.g., openapi-fetch)

### Negative
- Requires running `pnpm gen:api` when backend specs change (can be automated in CI)
- Fetch wrappers must be written manually for each endpoint (but they're small)
- Two sources of truth: generated types + hand-written fetch logic. Types stay in sync automatically, but fetch URLs/methods must be maintained manually.

### Neutral
- If the hand-written fetch wrappers become burdensome, migrating to a full generated client (openapi-fetch) is straightforward since the types are already generated
- The OpenAPI codegen pipeline is already established by ADR-009 and ADR-012
