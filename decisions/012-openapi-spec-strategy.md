# ADR-012: OpenAPI Spec Generation Strategy

## Status
Accepted

## Context
All BookieBreaker services expose REST APIs. ADR-009 established OpenAPI 3.1 code generation as the cross-service type sharing mechanism. We need to decide how OpenAPI specs are authored and maintained.

Options considered:

1. **Spec-first (manual):** Write OpenAPI YAML specs by hand in bookie-breaker-docs, then generate server stubs and clients. Maximum control, but specs can drift from implementation.
2. **Code-first with auto-generation:** Annotate server code and generate specs from it. Python/FastAPI does this automatically; Go/Echo requires middleware (e.g., swaggo/swag or huma).
3. **Hybrid:** Python services auto-generate (FastAPI's native OpenAPI); Go services use spec-first with oapi-codegen generating server interfaces from the spec.

## Decision
Use a **hybrid approach** matched to each language's strengths:

**Python services (FastAPI):** Code-first. FastAPI auto-generates OpenAPI 3.1 specs from Pydantic models and route decorators. The generated spec is the source of truth. Export it as a build artifact for client generation.

**Go services (Echo):** Spec-first. Write OpenAPI specs in `api-contracts/` in the docs repo. Use `oapi-codegen` to generate Go server interfaces and types from the spec. Implementation must conform to the generated interfaces — the spec is the source of truth and the implementation is validated at compile time.

**TypeScript (UI):** Consumer only. Generates TypeScript types from Python and Go service specs using `openapi-typescript`.

## Consequences

### Positive
- Python services get specs for free — FastAPI's auto-generation is accurate and always in sync with code
- Go services get compile-time validation — if the implementation drifts from the spec, it won't compile
- Both approaches produce valid OpenAPI 3.1 specs that feed into the codegen pipeline (ADR-009)
- Spec-first for Go encourages API design before implementation

### Negative
- Two different workflows (code-first vs spec-first) depending on the language
- Go developers must update the spec manually when changing the API, then re-run oapi-codegen
- FastAPI's auto-generated specs may include implementation details that leak into the public contract

### Neutral
- The docs repo `api-contracts/` directory serves as the canonical location for Go service specs
- Python service specs are extracted from running services (e.g., `GET /openapi.json`) and committed as artifacts
- CI validates that committed specs match what the services actually serve
