# ADR-013: Python Postgres Driver Selection

## Status

Accepted

## Context

Three Python services connect to PostgreSQL: prediction-engine, bookie-emulator, and (at Phase 6) the statistics-service
Python sidecar. All Python services use FastAPI with async request handling.

Options considered:

1. **asyncpg:** Purpose-built async Postgres driver. Fastest Python Postgres library. Native async/await support. Used
   by most FastAPI production deployments.
2. **psycopg3 (async mode):** The standard Python Postgres adapter, now with async support in v3. Broader ecosystem
   compatibility, more database features, slightly slower than asyncpg.

## Decision

Use **asyncpg** as the Postgres driver for all Python services.

asyncpg is the natural fit for FastAPI's async architecture. It provides native async connection pooling, prepared
statements, and binary protocol support for maximum throughput.

SQLAlchemy 2.0+ with its async engine can sit on top of asyncpg when an ORM layer is needed. Alembic continues to handle
migrations (Alembic runs synchronously, which is fine for migration scripts).

## Consequences

### Positive

- Best performance for async Postgres access — 2-5x faster than psycopg3 async for typical query patterns
- Native async/await integration with FastAPI's event loop
- Widely adopted in the FastAPI ecosystem — extensive community support and examples
- Built-in connection pooling suitable for service workloads

### Negative

- PostgreSQL-only — cannot be swapped for another database without changing the driver (not a concern for BookieBreaker)
- Some advanced Postgres features (COPY, LISTEN/NOTIFY) have different APIs than psycopg
- Alembic migrations run synchronously with a separate synchronous connection, adding a second dependency (but this is
  standard practice)

### Neutral

- SQLAlchemy async engine with asyncpg backend is a well-tested combination
- Connection pool configuration (min/max connections, timeouts) must be tuned per service based on workload
