# ADR-006: Best Tool Per Service (Polyglot Architecture)

## Status
Accepted

## Context
With 11 services, we needed to decide whether to standardize on a single language/framework or allow each service to use the best tool for its specific job.

A single language simplifies hiring (not applicable — solo project), code sharing, and cognitive load. Multiple languages let each service optimize for its domain: ML services benefit from Python's ecosystem, UIs benefit from TypeScript/React, compute-heavy services might benefit from Go or Rust.

## Decision
Select the best language, framework, and database for each service independently. Tech stack decisions will be made after requirements are locked in (see Phase 4 of the planning roadmap).

The primary constraint is pragmatism — as a solo developer, "best tool" must account for:
- Developer familiarity and velocity
- Ecosystem maturity for the service's domain
- Maintenance burden of supporting multiple languages
- Availability of libraries for key integrations

## Consequences

### Positive
- Each service uses the language/framework most natural for its domain
- No forcing square pegs into round holes (e.g., ML in Go, web UIs in Python)
- Services are truly independent — no shared runtime assumptions

### Negative
- Context switching between languages during development
- Harder to share code between services (must use API contracts, not shared libraries, unless languages align)
- More diverse tooling to maintain (linters, test frameworks, CI pipelines per language)

### Neutral
- In practice, a solo developer will likely converge on 2-3 languages maximum
- The shared code strategy ([ADR-009](009-shared-code-strategy.md)) further constrains this decision
