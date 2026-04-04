# ADR-019: Database Migration Tooling

## Status
Accepted

## Context
Three services own database schemas in PostgreSQL: lines-service (Go), prediction-engine (Python), and bookie-emulator (Python). Each needs a migration framework to create and evolve its schema.

Options per language:

**Go:**
1. **golang-migrate:** Standalone migration tool. SQL-based migrations. No ORM dependency. Well-maintained.
2. **goose:** Similar to golang-migrate, supports both SQL and Go-based migrations. Slightly less popular.
3. **Custom:** Hand-rolled migration runner. Maximum control, maintenance burden.

**Python:**
1. **Alembic:** SQLAlchemy's migration tool. Auto-generates migrations from model changes. Industry standard for Python/SQLAlchemy projects.
2. **Custom:** Hand-rolled migration runner.

## Decision
- **Go services:** Use **golang-migrate** with SQL migration files
- **Python services:** Use **Alembic** with SQLAlchemy model-driven migrations

**golang-migrate** is the most popular Go migration tool. It uses plain SQL files (`001_create_tables.up.sql` / `001_create_tables.down.sql`), keeping migrations database-native with no Go code required. It supports PostgreSQL natively and can be run as a CLI tool or embedded in the Go binary.

**Alembic** is the standard migration tool for Python projects using SQLAlchemy. It auto-generates migration scripts by diffing SQLAlchemy model definitions against the current database state, reducing manual migration writing.

Migration files live in each service repo:
- Go: `migrations/` directory with numbered `.up.sql` / `.down.sql` pairs
- Python: `alembic/` directory with Alembic's standard structure

The root `Taskfile.yml` orchestrates migrations across all services via `task db:migrate`.

## Consequences

### Positive
- Both tools are the standard choice for their respective languages — maximum community support
- golang-migrate's SQL files are database-native — no Go ORM coupling
- Alembic's auto-generation reduces manual work when Python models change
- Both support up/down migrations for reversibility

### Negative
- Two different migration tools to learn and maintain
- golang-migrate requires writing SQL by hand (no auto-generation from Go structs)
- Alembic auto-generation occasionally produces incorrect migrations for complex schema changes — manual review required

### Neutral
- Migration ordering across services is handled by the root Taskfile (lines-service first, then prediction-engine, then bookie-emulator — matching the schema dependency order)
- Both tools support running migrations in CI for integration tests
