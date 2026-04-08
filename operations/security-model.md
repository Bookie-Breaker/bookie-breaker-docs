# Security Model

Security posture for BookieBreaker. The system handles no user accounts or PII in v1 -- the primary assets to protect are external API keys and the integrity of prediction/paper-trading data.

---

## 1. External API Key Management

### Services Requiring External API Keys

| Service       | External API  | Key Environment Variable | Purpose                                             |
| ------------- | ------------- | ------------------------ | --------------------------------------------------- |
| lines-service | The Odds API  | `ODDS_API_KEY`           | Betting line ingestion from 40+ sportsbooks         |
| lines-service | SharpAPI      | `SHARP_API_KEY`          | Real-time line movement via SSE stream              |
| agent         | Anthropic API | `ANTHROPIC_API_KEY`      | LLM-powered analysis and natural language synthesis |

These are the only services that make authenticated calls to external APIs. statistics-service uses Python packages (nfl_data_py, nba_api, pybaseball, CFBD) that access public endpoints and do not require API keys.

### Key Storage

**Development:** API keys are stored in `.env` files at the project root, loaded by Docker Compose. Each repo's `.env` file is listed in `.gitignore` and never committed.

```bash
# .env (never committed)
ODDS_API_KEY=live_abc123...
SHARP_API_KEY=sk_def456...
ANTHROPIC_API_KEY=sk-ant-...
```

**Production (Kubernetes):** API keys are stored as Kubernetes Secrets, mounted as environment variables into the relevant pods. See section 2 for details.

### Key Rotation Strategy

| API           | Rotation Frequency         | Rotation Process                                                                 |
| ------------- | -------------------------- | -------------------------------------------------------------------------------- |
| The Odds API  | On compromise or annual    | Generate new key in dashboard, update `.env` / K8s Secret, restart lines-service |
| SharpAPI      | On compromise or annual    | Same process                                                                     |
| Anthropic API | On compromise or quarterly | Generate new key in Anthropic Console, update `.env` / K8s Secret, restart agent |

**Rotation steps:**

1. Generate new key in the provider's dashboard
2. Update the `.env` file (development) or Kubernetes Secret (production)
3. Restart the affected service (`docker compose restart <service>` or rolling deployment)
4. Verify the service is healthy via `/health` endpoint
5. Revoke the old key in the provider's dashboard

No zero-downtime rotation is needed in v1. Services are stateless and restart in seconds.

---

## 2. Secrets Management

### Development Environment

Every repository that uses secrets includes a `.env.example` file with placeholder values and comments explaining each variable. The actual `.env` file is created by copying `.env.example` and filling in real values.

```bash
# .env.example (committed to git)
# Lines-service external API keys
ODDS_API_KEY=your_odds_api_key_here
SHARP_API_KEY=your_sharp_api_key_here

# Database credentials
LINES_DB_USER=lines_service
LINES_DB_PASSWORD=change_me_in_dot_env
LINES_DB_NAME=lines
LINES_DB_HOST=lines-db
LINES_DB_PORT=5432

# Redis
REDIS_URL=redis://redis:6379
```

**Rules:**

- `.env` is always in `.gitignore`. Every repo's `.gitignore` includes `.env` and `*.env.local`.
- `.env.example` contains no real secrets -- only placeholder strings and documentation.
- Taskfile commands that bootstrap a new development environment copy `.env.example` to `.env` if `.env` does not exist.

### Production Environment (Kubernetes)

```yaml
# Kubernetes Secret (applied manually or via sealed-secrets/external-secrets-operator)
apiVersion: v1
kind: Secret
metadata:
  name: lines-service-secrets
  namespace: bookiebreaker
type: Opaque
data:
  ODDS_API_KEY: <base64-encoded>
  SHARP_API_KEY: <base64-encoded>
  LINES_DB_PASSWORD: <base64-encoded>
```

**Future upgrade path:** If the project moves beyond single-host deployment, integrate an external secrets manager (HashiCorp Vault, AWS Secrets Manager, or GCP Secret Manager) via the Kubernetes External Secrets Operator. For v1, native Kubernetes Secrets are sufficient.

### Database Credentials

Each service that owns a database schema connects with a dedicated database user that has permissions only on its own schema. No service uses the PostgreSQL superuser.

| Service            | DB User              | Schema        | Permissions                 |
| ------------------ | -------------------- | ------------- | --------------------------- |
| lines-service      | `lines_service`      | `lines`       | ALL on `lines` schema       |
| prediction-engine  | `prediction_service` | `predictions` | ALL on `predictions` schema |
| bookie-emulator    | `bookie_service`     | `bookie`      | ALL on `bookie` schema      |
| agent              | `agent_service`      | `agent`       | ALL on `agent` schema       |
| simulation-engine  | `simulation_service` | `simulations` | ALL on `simulations` schema |
| statistics-service | `stats_service`      | `stats`       | ALL on `stats` schema       |

Database user creation is handled by initialization scripts in each service's database container (`docker-entrypoint-initdb.d/`).

---

## 3. Inter-Service Authentication

### Phase 1: Trust the Network (Current)

All services run on an internal Docker network (`bookiebreaker`). No inter-service authentication is required because:

- Services are not exposed to the public internet (only ui:3000, agent:8006, and mcp-server:8007 are mapped to host ports)
- The Docker network provides process-level isolation -- only containers on the `bookiebreaker` network can reach internal services
- There is a single operator (solo developer) with full access to the host machine

**Trust boundary:** Everything inside the Docker Compose network is trusted. Anything outside (the host network, the internet) is untrusted.

```
                    +----- Trust Boundary -----+
                    |                          |
  Internet  -----X-|-> ui (3000)              |
                    |-> agent (8006)           |
                    |-> mcp-server (8007)      |
                    |                          |
                    |   lines-service (8001)   |  <-- internal only
                    |   statistics-svc (8002)  |  <-- internal only
                    |   simulation-eng (8003)  |  <-- internal only
                    |   prediction-eng (8004)  |  <-- internal only
                    |   bookie-emu (8005)      |  <-- internal only
                    |   redis (6379)           |  <-- internal only
                    |   postgres (5432-5437)   |  <-- internal only
                    +--------------------------+
```

### Phase 2: API Key Authentication (Future)

If services are exposed beyond the local Docker network (e.g., multi-host deployment, cloud hosting), add a shared API key for inter-service calls:

- Each service validates an `X-API-Key` header on incoming requests
- The shared key is distributed via the secrets management system (environment variables / Kubernetes Secrets)
- Middleware in Echo (Go) and FastAPI (Python) handles validation

```python
# Python example (FastAPI middleware)
@app.middleware("http")
async def verify_api_key(request: Request, call_next):
    if request.url.path == "/health" or request.url.path == "/metrics":
        return await call_next(request)
    api_key = request.headers.get("X-API-Key")
    if api_key != settings.INTERNAL_API_KEY:
        return JSONResponse(status_code=401, content={"error": "Unauthorized"})
    return await call_next(request)
```

### Phase 3: JWT-Based Auth (Future, if Needed)

If BookieBreaker adds user accounts or multi-tenant access, introduce JWT-based authentication with per-user tokens. This is not planned for v1.

---

## 4. Network Security

### Docker Network Isolation

The `docker-compose.yml` defines a single bridge network. Only explicitly mapped ports are accessible from the host.

```yaml
networks:
  bookiebreaker:
    driver: bridge

services:
  # Exposed to host
  ui:
    ports:
      - "3000:3000"
  agent:
    ports:
      - "8006:8006"
  mcp-server:
    ports:
      - "8007:8007"

  # Internal only (no ports mapping)
  lines-service:
    expose:
      - "8001"
  statistics-service:
    expose:
      - "8002"
  # ... etc.
```

**Key rules:**

- Database containers never expose ports to the host. Access PostgreSQL only via `docker compose exec` for debugging.
- Redis never exposes ports to the host. No external Redis clients.
- Internal services use `expose` (makes port available within Docker network) but not `ports` (which maps to host).

### Production: Kubernetes NetworkPolicies

When deploying to Kubernetes, define NetworkPolicies that restrict which pods can communicate:

```yaml
# Example: lines-service can only be reached by agent, simulation-engine,
# prediction-engine, and bookie-emulator
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: lines-service-ingress
  namespace: bookiebreaker
spec:
  podSelector:
    matchLabels:
      app: lines-service
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: agent
        - podSelector:
            matchLabels:
              app: prediction-engine
        - podSelector:
            matchLabels:
              app: bookie-emulator
        - podSelector:
            matchLabels:
              app: ui
        - podSelector:
            matchLabels:
              app: cli
```

### Outbound Access

Only three services require outbound internet access:

| Service            | Destination                               | Purpose                    |
| ------------------ | ----------------------------------------- | -------------------------- |
| lines-service      | api.the-odds-api.com, sharp API endpoints | Line ingestion             |
| statistics-service | Various stats API endpoints               | Statistical data ingestion |
| agent              | api.anthropic.com                         | LLM analysis               |

In Kubernetes, an egress NetworkPolicy can restrict outbound traffic to only these known endpoints.

---

## 5. Data Security

### Data Classification

| Data Category        | Sensitivity | Examples                                          | Protection                                                                              |
| -------------------- | ----------- | ------------------------------------------------- | --------------------------------------------------------------------------------------- |
| API keys             | **High**    | ODDS_API_KEY, ANTHROPIC_API_KEY                   | Encrypted at rest (K8s Secrets), never logged, never committed to git                   |
| Database credentials | **High**    | DB passwords                                      | Same as API keys                                                                        |
| Prediction data      | **Medium**  | Calibrated probabilities, edges, model parameters | Integrity important (no unauthorized modification), no encryption at rest needed for v1 |
| Paper trading data   | **Medium**  | Bankroll, bet history, ROI                        | Integrity important for validation accuracy, no encryption at rest needed for v1        |
| Betting lines        | **Low**     | Market odds from public APIs                      | Publicly available data, no special protection                                          |
| Statistics           | **Low**     | Team stats, game results                          | Publicly available data, no special protection                                          |

### What is NOT Stored

- No PII (no user accounts, no personal data in v1)
- No real money transactions (paper trading only)
- No payment information
- No authentication tokens for end users

### Key Protection Measures

- **API keys must never appear in logs.** Log middleware must redact or omit any header or field containing key material. Structured logging makes this straightforward -- never log the full request headers.
- **API keys must never appear in error messages.** If an external API returns a 401, log "External API authentication failed" not the key itself.
- **Database backups** (if taken) should be stored with appropriate access controls. For v1, Docker volume snapshots are sufficient.

---

## 6. Dependency Security

### Automated Dependency Updates

All repositories use Renovate (preferred) or Dependabot for automated dependency update PRs. See [CI/CD GitHub](ci-cd-github.md) for Renovate configuration details.

### Audit Commands in CI

Every CI pipeline includes a dependency audit step that fails the build if known vulnerabilities are found.

| Language   | Audit Command                        | Notes                                                                                     |
| ---------- | ------------------------------------ | ----------------------------------------------------------------------------------------- |
| Go         | `go mod verify && govulncheck ./...` | `govulncheck` checks the Go vulnerability database                                        |
| Python     | `pip audit`                          | Checks the PyPI advisory database. Run via `uv run pip-audit`.                            |
| TypeScript | `pnpm audit --audit-level=high`      | Fails on high or critical vulnerabilities only (low/moderate are tracked but don't block) |

### Container Image Scanning

All Docker images are scanned for OS-level and application-level vulnerabilities before deployment.

```yaml
# GitHub Actions step (included in reusable workflow)
- name: Scan container image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.IMAGE_NAME }}:${{ github.sha }}
    format: "sarif"
    output: "trivy-results.sarif"
    severity: "HIGH,CRITICAL"
    exit-code: "1" # Fail the build on high/critical vulnerabilities
```

**Base image policy:**

- Go services: `gcr.io/distroless/static-debian12` (minimal attack surface, no shell)
- Python services: `python:3.12-slim` (slim variant, not full Debian)
- TypeScript/SvelteKit: `node:22-slim` for build, `node:22-slim` for runtime
- Rebuild images weekly to pick up base image security patches (automated via Renovate's Docker image digest tracking)

### Supply Chain Protections

- **Lock files committed:** `go.sum`, `uv.lock`, `pnpm-lock.yaml` are always committed and verified in CI.
- **Checksum verification:** Go modules are verified via `go mod verify`. Python and Node dependencies are verified by their respective lock file checksums.
- **Minimal dependencies:** Prefer standard library solutions where feasible. Each new dependency is a conscious choice, not an automatic addition.

---

## Security Checklist for New Services

When adding a new service to BookieBreaker, verify:

- [ ] `.env.example` exists with all required environment variables (no real secrets)
- [ ] `.env` is in `.gitignore`
- [ ] Structured logging does not emit API keys or credentials
- [ ] `/health` endpoint does not expose sensitive information
- [ ] Docker Compose does not map ports to host unless external access is required
- [ ] Database user has minimal permissions (own schema only)
- [ ] CI pipeline includes dependency audit step
- [ ] Docker image uses a minimal base image
- [ ] Trivy scan is included in the CI pipeline
- [ ] Any new external API keys are documented in this file
