# ADR-005: Containerized Deployment (Docker/K8s)

## Status
Accepted

## Context
With 11 services, the system needs a consistent deployment and orchestration strategy. Options ranged from bare-metal/VM deployment to fully managed cloud services to containerized deployment.

## Decision
All services are containerized with Docker. The system runs anywhere containers run:
- **Local development:** docker-compose for the full stack
- **Production (optional):** Kubernetes or any container orchestration platform

Each service has its own Dockerfile. A root docker-compose.yml in `bookie-breaker-infra-ops` defines the full local stack including databases, message brokers, and all services.

## Consequences

### Positive
- Environment parity between dev and prod — "works on my machine" issues are minimized
- Easy to spin up the full stack or individual services locally
- Cloud-agnostic — not locked into AWS, GCP, or Azure
- Each service is independently deployable

### Negative
- Docker adds overhead for local development (memory, CPU, startup time)
- Kubernetes adds significant operational complexity if/when needed
- 11 containers running simultaneously may strain a development machine

### Neutral
- Cloud deployment is optional — the system can run entirely on a single machine via docker-compose
- The infra-ops repo owns all deployment configuration
