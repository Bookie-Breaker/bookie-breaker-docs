# ADR-015: Pipeline Scheduler Selection

## Status

Accepted

## Context

The agent service orchestrates the prediction pipeline: fetch stats → run simulations → generate predictions → detect edges. This pipeline needs to run on a schedule (e.g., "2 hours before the first NBA game each day") and on-demand via API.

Options considered:

1. **APScheduler:** Mature Python scheduling library. Supports cron, interval, and date-based triggers. Can persist job state to a database. Well-tested with async frameworks.
2. **Custom scheduler:** Simple `asyncio` task loop with sleep intervals. Minimal dependency, full control, but requires implementing cron parsing, missed job handling, and persistence from scratch.
3. **External scheduler (cron, Kubernetes CronJob):** Offload scheduling to infrastructure. Clean separation, but adds operational complexity and makes the schedule harder to configure at runtime.

## Decision

Use **APScheduler** (v4.x with async support) for pipeline scheduling within the agent service.

APScheduler provides:

- Cron-style triggers with timezone awareness (essential for sports — game times are in local timezones)
- Async job execution that integrates with FastAPI's event loop
- Job persistence via PostgreSQL (using the agent's existing database connection) so schedules survive restarts
- Runtime schedule modification via API (add/remove/pause jobs without redeployment)

On-demand pipeline runs are triggered via `POST /api/v1/pipeline/run` and bypass the scheduler entirely.

## Consequences

### Positive

- Proven library with 10+ years of production use
- Cron expressions handle complex schedules ("2 hours before first game" can be computed daily and scheduled dynamically)
- Job persistence means schedules survive agent restarts
- Async-native in v4.x — no thread pool overhead

### Negative

- Adds a dependency (APScheduler + its data store adapter)
- APScheduler v4.x is a significant rewrite from v3.x — less community precedent for the async API
- Job persistence requires a database table in the agent's schema

### Neutral

- If APScheduler proves problematic, migrating to a custom asyncio scheduler is straightforward — the scheduling logic is encapsulated in the agent service
- External schedulers (Kubernetes CronJob) remain viable for production deployment if runtime configurability isn't needed
