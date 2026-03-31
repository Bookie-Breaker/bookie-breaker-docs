# bookie-breaker-infra-ops

## Purpose

Owns all infrastructure, deployment, and operational configuration for the BookieBreaker system. Contains Docker configurations, docker-compose for local development, CI/CD pipelines, and shared GitHub organization settings.

## Responsibilities

- Owns Dockerfiles and docker-compose configurations for local development of the full system.
- Owns CI/CD pipeline definitions (GitHub Actions workflows) for all repos.
- Owns deployment configurations (staging, production) and environment-specific settings.
- Owns shared GitHub organization settings: issue templates, PR templates, Renovate configuration, branch protection rules, CODEOWNERS.
- Defines and maintains the service dependency graph for deployment ordering.
- Owns monitoring, logging, and alerting infrastructure configuration (Prometheus, Grafana, or equivalent).
- Manages secrets management configuration (not the secrets themselves).
- Owns database migration tooling and orchestration.

## Non-Responsibilities

- Does NOT contain application code or business logic for any service.
- Does NOT own API contracts or data schemas. Those belong to the individual services.
- Does NOT own documentation content. That belongs to bookie-breaker-docs.
- Does NOT manage external API keys or credentials directly -- it configures the vaults/secret stores where they live.
- Does NOT make decisions about what gets deployed. It provides the mechanism; teams decide the policy.

## Inputs

| Source | Data | Mechanism |
|---|---|---|
| All service repos | Dockerfiles, build artifacts, test results | Git / CI triggers |
| GitHub | Push events, PR events, schedule triggers | GitHub Actions webhooks |
| Developers | Infrastructure change requests | Pull requests to this repo |
| External services | Health checks, metrics | Monitoring endpoints |

## Outputs

| Destination | Data | Mechanism |
|---|---|---|
| All services | Built and deployed containers | Docker registry / deployment pipeline |
| Developers | Local development environment (docker-compose up) | Docker Compose |
| GitHub | CI/CD status checks, workflow results | GitHub Actions |
| Monitoring stack | Dashboards, alert rules | Infrastructure-as-code configs |

## Dependencies

- **All service repos** -- needs their Dockerfiles and build configurations
- **GitHub** (external) -- CI/CD execution platform
- **Docker registry** (external) -- container image storage

## Dependents

- **All services** -- every service depends on infra-ops for its build, test, and deployment pipeline
- **Developers** -- depend on docker-compose for local development
