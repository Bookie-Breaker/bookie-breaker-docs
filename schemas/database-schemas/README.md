# Database Schema Overview

Database schemas for BookieBreaker, a distributed sports prediction system. All schemas live in a single PostgreSQL 16 instance with the TimescaleDB extension enabled. Redis 7 provides caching, pub/sub, and ephemeral storage.

---

## Database Topology

### PostgreSQL 16 + TimescaleDB

A single PostgreSQL instance hosts three isolated schemas, one per data-owning service:

| Schema        | Owner Service     | Storage Type             | Purpose                                                                           |
| ------------- | ----------------- | ------------------------ | --------------------------------------------------------------------------------- |
| `lines`       | lines-service     | TimescaleDB hypertables  | Line snapshots with time-series partitioning, compression, and retention policies |
| `predictions` | prediction-engine | Standard Postgres tables | Predictions, model versions, feature vectors                                      |
| `emulator`    | bookie-emulator   | Standard Postgres tables | Paper bets, grades, bankroll snapshots, performance summaries                     |

Each service connects with a dedicated Postgres role that has full access to its own schema and no access to other schemas. Cross-service data access happens exclusively through REST APIs, never through direct database queries.

**Services with NO Postgres tables:**

- statistics-service -- Redis cache only; external APIs are the source of truth
- simulation-engine -- Redis cache only; results are ephemeral
- agent -- Redis cache only; edges and analyses are transient

### Redis 7

Redis serves four purposes across the system:

| Purpose           | Services           | Details                                                         |
| ----------------- | ------------------ | --------------------------------------------------------------- |
| Statistics cache  | statistics-service | Team stats, player stats, game data, feature vectors, schedules |
| Simulation cache  | simulation-engine  | Simulation results and distribution data                        |
| Agent cache       | agent              | Dashboard snapshots, analysis text                              |
| Pub/Sub event bus | All services       | Decoupled event-driven communication between services           |

See [redis-schemas.md](redis-schemas.md) for full key patterns, TTLs, and event payload schemas.

---

## Naming Conventions

All database objects follow these conventions:

| Rule               | Convention                       | Example                                         |
| ------------------ | -------------------------------- | ----------------------------------------------- |
| Table names        | snake_case, plural               | `line_snapshots`, `paper_bets`                  |
| Column names       | snake_case                       | `game_external_id`, `created_at`                |
| Primary keys       | `id` (UUID v4)                   | `id UUID PRIMARY KEY DEFAULT gen_random_uuid()` |
| Foreign keys       | `{referenced_table_singular}_id` | `sportsbook_id`, `model_version_id`             |
| Indexes            | `idx_{table}_{columns}`          | `idx_line_snapshots_game_market`                |
| Unique constraints | `uq_{table}_{columns}`           | `uq_line_snapshots_composite`                   |
| Check constraints  | `chk_{table}_{description}`      | `chk_predictions_probability_range`             |
| Enums (Postgres)   | `{name}_enum`                    | `league_enum`, `market_type_enum`               |
| Timestamps         | `timestamptz` (always UTC)       | `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()` |
| Booleans           | `is_` prefix                     | `is_active`, `is_live`                          |
| JSONB columns      | Descriptive noun                 | `evaluation_metrics`, `features`                |

---

## Shared Enum Types

These Postgres enum types are created once and used across schemas:

```sql
-- Shared enums (created in public schema, referenced by all service schemas)

CREATE TYPE league_enum AS ENUM (
    'NFL', 'NBA', 'MLB', 'NCAA_FB', 'NCAA_BB', 'NCAA_BSB'
);

CREATE TYPE market_type_enum AS ENUM (
    'SPREAD', 'TOTAL', 'MONEYLINE', 'PLAYER_PROP', 'TEAM_PROP', 'GAME_PROP', 'FUTURE', 'LIVE'
);

CREATE TYPE sport_enum AS ENUM (
    'FOOTBALL', 'BASKETBALL', 'BASEBALL'
);

CREATE TYPE bet_result_enum AS ENUM (
    'OPEN', 'WON', 'LOST', 'PUSH', 'VOID'
);
```

---

## Migration Strategy

### Tool: golang-migrate or Alembic

Each service manages its own migrations within its schema. Migrations are stored in the service's repository under `migrations/` and run at service startup or via a dedicated migration command.

### Principles

1. **Forward-only migrations.** Down migrations are maintained for development convenience but never run in production. Rollback is done by deploying a new forward migration that reverses the change.
2. **One migration per change.** Each schema change gets its own numbered migration file. No multi-change migration files.
3. **Schema-qualified.** All migrations explicitly set `search_path` to the service's schema to prevent accidental cross-schema changes.
4. **Idempotent when possible.** Use `IF NOT EXISTS` / `IF EXISTS` where SQL supports it.
5. **Data migrations are separate.** Schema changes and data backfills are separate migration files. Schema first, data second.
6. **TimescaleDB awareness.** Hypertable creation and compression/retention policy changes go in migrations. Chunk interval changes require careful planning as they only affect future chunks.

### Migration numbering

```
migrations/
  001_create_schema.sql
  002_create_enums.sql
  003_create_sportsbooks.sql
  004_create_line_snapshots.sql
  ...
```

---

## Indexing Principles

1. **Index for query patterns, not for columns.** Every index must map to a known API endpoint or query pattern documented in the component spec.
2. **Composite indexes follow query selectivity.** Most selective column first (equality), then range/sort columns.
3. **Covering indexes for hot paths.** For the most frequent queries, include all selected columns in the index to enable index-only scans.
4. **TimescaleDB chunk exclusion.** For hypertables, the time column is always the last element of composite indexes because TimescaleDB already partitions by time.
5. **Partial indexes for enum filters.** Use `WHERE` clauses on partial indexes for common filtered queries (e.g., `WHERE status = 'OPEN'` on paper_bets).
6. **No premature indexing.** Start with the indexes listed in each schema document. Add more based on `EXPLAIN ANALYZE` of slow queries in production.
7. **Monitor bloat.** Schedule regular `REINDEX CONCURRENTLY` for high-write tables (line_snapshots, predictions).

---

## Schema Documents

- [lines-service.md](lines-service.md) -- TimescaleDB hypertable schema for betting line snapshots
- [prediction-engine.md](prediction-engine.md) -- Standard tables for predictions and model management
- [bookie-emulator.md](bookie-emulator.md) -- Standard tables for paper trading and performance tracking
- [redis-schemas.md](redis-schemas.md) -- Redis key patterns, TTLs, and pub/sub event payloads

---

## Estimated Storage

| Schema                        | Year 1 Rows | Year 1 Size (est.)           | Year 3 Size (est.)              |
| ----------------------------- | ----------- | ---------------------------- | ------------------------------- |
| `lines.line_snapshots`        | 5-10M       | 2-5 GB (compressed: ~500 MB) | 15-30M rows, ~1.5 GB compressed |
| `lines.closing_lines`         | ~100K       | ~50 MB                       | ~300K rows, ~150 MB             |
| `predictions.predictions`     | ~100-500K   | ~200 MB                      | ~1-1.5M rows, ~600 MB           |
| `predictions.feature_vectors` | ~100-500K   | ~500 MB                      | ~1-1.5M rows, ~1.5 GB           |
| `emulator.paper_bets`         | ~5-20K      | ~20 MB                       | ~15-60K rows, ~60 MB            |
| `emulator.bankroll_snapshots` | ~1-5K       | ~5 MB                        | ~3-15K rows, ~15 MB             |

Total uncompressed: ~3-6 GB/year. With TimescaleDB compression on line_snapshots: ~1-2 GB/year. Storage is modest and a single-instance Postgres handles this comfortably.
