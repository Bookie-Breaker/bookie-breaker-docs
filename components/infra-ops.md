# bookie-breaker-infra-ops

## Purpose

Owns all infrastructure, deployment, and operational configuration for the BookieBreaker system. Contains Docker configurations, docker-compose for local development, CI/CD pipelines, and shared GitHub organization settings.

## Responsibilities

- Owns Dockerfiles and docker-compose configurations for local development of the full system.
- Owns CI/CD pipeline definitions (GitHub Actions workflows) for all repos.
- Owns deployment configurations (staging, production) and environment-specific settings.
- Owns shared GitHub organization settings: issue templates, PR templates, Renovate configuration, branch protection rules, CODEOWNERS.
- Defines and maintains the service dependency graph for deployment ordering.
- Owns the OpenTelemetry-based observability stack: otel-collector, Grafana, Prometheus (metrics), Tempo (traces), and Loki (logs). All services export OTLP telemetry from day one.
- Owns local LLM infrastructure: Ollama container in Docker Compose for self-hosted model serving, model pull scripts, and GPU passthrough configuration. See [ADR-011](../decisions/011-local-llm-strategy.md).
- Manages secrets across the full lifecycle: development uses `.env` files (with `.env.example` templates committed per repo); production uses Kubernetes Secrets with an upgrade path to External Secrets Operator + HashiCorp Vault. Documents secret rotation procedures and ensures API keys never appear in logs or error messages. See [Security Model](../operations/security-model.md) for full details.
- Owns database migration tooling and orchestration.
- Owns shared developer tooling configuration: mise tool versions, lefthook git hooks, commitlint, markdownlint, yamllint, and other linter/formatter configs across all repos (see [Repo Standards](../operations/repo-standards.md), [Tool Management](../operations/tool-management.md), [Git Hooks](../operations/git-hooks.md)).

## Non-Responsibilities

- Does NOT contain application code or business logic for any service.
- Does NOT own API contracts or data schemas. Those belong to the individual services.
- Does NOT own documentation content. That belongs to bookie-breaker-docs.
- Does NOT manage external API keys or credentials directly -- it configures the secret stores and `.env.example` templates where they live. Actual secret values are managed by the developer (dev) or Kubernetes Secrets / Vault (prod).
- Does NOT make decisions about what gets deployed. It provides the mechanism; teams decide the policy.

## Inputs

| Source            | Data                                       | Mechanism                  |
| ----------------- | ------------------------------------------ | -------------------------- |
| All service repos | Dockerfiles, build artifacts, test results | Git / CI triggers          |
| GitHub            | Push events, PR events, schedule triggers  | GitHub Actions webhooks    |
| Developers        | Infrastructure change requests             | Pull requests to this repo |
| External services | Health checks, metrics                     | Monitoring endpoints       |

## Outputs

| Destination      | Data                                              | Mechanism                             |
| ---------------- | ------------------------------------------------- | ------------------------------------- |
| All services     | Built and deployed containers                     | Docker registry / deployment pipeline |
| Developers       | Local development environment (docker-compose up) | Docker Compose                        |
| GitHub           | CI/CD status checks, workflow results             | GitHub Actions                        |
| Monitoring stack | Dashboards, alert rules                           | Infrastructure-as-code configs        |

## Dependencies

- **All service repos** -- needs their Dockerfiles and build configurations
- **GitHub** (external) -- CI/CD execution platform
- **Docker registry** (external) -- container image storage

## Dependents

- **All services** -- every service depends on infra-ops for its build, test, and deployment pipeline
- **Developers** -- depend on docker-compose for local development

---

## Requirements

### Functional Requirements

- **FR-001:** Provide a single `docker-compose.yml` that starts all 10 application services, 6 per-service PostgreSQL databases, and the shared Redis instance with correct networking, volume mounts, and environment variables.
- **FR-002:** Support `docker compose up` for full local development with all services running on a shared Docker bridge network (`bookiebreaker`), with inter-service DNS resolution by container name.
- **FR-003:** Provide individual Dockerfiles for each service, optimized for build caching (multi-stage builds where appropriate, dependency layers cached separately from application code).
- **FR-004:** Define CI/CD workflows (GitHub Actions) for all 10 service repos: lint, test, build Docker image, and push to container registry on merge to main.
- **FR-005:** Provide environment-specific configuration templates for local development, staging, and production, including all service URLs, database connection strings, Redis URLs, and external API keys (as placeholder references).
- **FR-006:** Define and document the service dependency graph for deployment ordering: databases and Redis must start before data-layer services; data-layer services before compute-layer; compute-layer before orchestration-layer; orchestration-layer before interface-layer.
- **FR-007:** Provide health check configurations for all services in Docker Compose, ensuring dependent services wait for their dependencies to be healthy before starting.
- **FR-008:** Manage database migration tooling: provide a standardized migration framework (e.g., Alembic for Python services) and orchestration scripts to run migrations in the correct order across all services.
- **FR-009:** Configure the OpenTelemetry observability stack: otel-collector receives OTLP from all services, forwards metrics to Prometheus, traces to Tempo, and logs to Loki. Grafana dashboards visualize all three signals. All deployed components must export OTEL telemetry from Phase 1. See [ADR-012](../decisions/012-observability-otel-first.md).
- **FR-010:** Configure centralized logging via OTEL logs pipeline: all services emit structured JSON logs, otel-collector ships them to Loki for centralized querying through Grafana.
- **FR-011:** Provide shared GitHub organization configuration: issue templates, PR templates, branch protection rules (require CI pass before merge), CODEOWNERS for each repo, and Renovate configuration for dependency updates.
- **FR-012:** Manage secrets configuration: define which secrets each service needs, provide `.env.example` templates for development, Kubernetes Secrets integration for production (upgrade path to External Secrets Operator + Vault), and document secret rotation procedures.
- **FR-015:** Provide local LLM infrastructure: Ollama container in Docker Compose with configurable model pull on first start (default: a small model for dev), GPU passthrough support via `docker-compose.gpu.yml` overlay, and health check for model readiness.
- **FR-016:** (Future, Phase 5+) Evaluate kagent (CNCF Sandbox) for Kubernetes-native agent orchestration with built-in chat UI and native MCP server support. See [ADR-010](../decisions/010-tech-stack-selection.md) for evaluation notes.
- **FR-013:** Provide scripts for common operational tasks: full system start/stop, database backup/restore, log aggregation, and service restart.
- **FR-014:** Define resource limits (CPU, memory) for each container in Docker Compose based on expected workload profiles (simulation-engine needs more CPU; databases need more memory).

### Non-Functional Requirements

- **Latency:** `docker compose up` for full system start: < 3 minutes (cold start with image builds), < 30 seconds (warm start with cached images). CI/CD pipeline for a single service: < 10 minutes from push to image published.
- **Throughput:** Support 1 developer running the full stack locally. CI/CD handles up to 10 concurrent pipeline runs across repos (GitHub Actions concurrency).
- **Availability:** Local development environment: best-effort (developer restarts on failure). Production deployment: health checks enable automatic container restart on failure. Monitoring alerts on any service being down for > 5 minutes.
- **Storage:** Docker volumes for each database: initial ~1 GB each, growing per service-specific estimates. Redis volume: < 500 MB. Container images: ~200-500 MB each. Total disk footprint for full local stack: ~10-20 GB.

### Data Ownership

None. infra-ops does not own any domain entities. It provides the infrastructure and deployment mechanism for all other services.

### APIs Exposed

None. infra-ops is not a runtime service. It does not expose APIs.

### APIs Consumed

None. infra-ops does not consume APIs at runtime. It configures the infrastructure that enables API communication between services.

### Events Published

None.

### Events Subscribed

None.

### Storage Requirements

- **Repository contents:** Docker Compose files, Dockerfiles, GitHub Actions workflows, environment templates, monitoring configs, migration scripts. Total: < 50 MB.
- **No runtime database.** infra-ops does not have its own database.
- **Provisioned infrastructure storage:**
  - 6 PostgreSQL database volumes: ~1-5 GB each initially, growing per service estimates.
  - 1 Redis volume: < 500 MB (cache + pub/sub state).
  - Docker image cache: ~5-10 GB.
  - Log storage: depends on retention policy; estimate ~1 GB/month for all services combined.
  - Monitoring data (Prometheus): ~2-5 GB with 30-day retention.
  - Trace data (Tempo): ~1-3 GB with 7-day retention.
  - Log data (Loki): ~2-5 GB with 30-day retention.
  - LLM model files (Ollama): ~2-8 GB depending on model selection.
