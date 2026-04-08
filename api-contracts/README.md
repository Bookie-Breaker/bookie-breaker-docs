# API Design Principles

Standards and conventions that apply to all BookieBreaker REST APIs. Every service follows these rules to ensure
consistency across Go (Echo) and Python (FastAPI) services.

---

## URL Structure

All endpoints follow this pattern:

```text
/api/v1/{service}/{resource}
```

The service short name is always present as a path component:

| Service            | Short Name | Base URL           |
| ------------------ | ---------- | ------------------ |
| lines-service      | `lines`    | `/api/v1/lines`    |
| statistics-service | `stats`    | `/api/v1/stats`    |
| simulation-engine  | `sim`      | `/api/v1/sim`      |
| prediction-engine  | `predict`  | `/api/v1/predict`  |
| bookie-emulator    | `emulator` | `/api/v1/emulator` |
| agent              | `agent`    | `/api/v1/agent`    |

Within Docker Compose, full base URLs resolve as:

```text
http://lines-service:8001/api/v1/lines
http://statistics-service:8002/api/v1/stats
http://simulation-engine:8003/api/v1/sim
http://prediction-engine:8004/api/v1/predict
http://bookie-emulator:8005/api/v1/emulator
http://agent:8006/api/v1/agent
```

---

## Versioning

API versions are embedded in the URL path (`/v1/`). The version number increments only on breaking changes. Non-breaking
additions (new optional fields, new endpoints) do not require a version bump.

When a breaking change is necessary, both `/v1/` and `/v2/` are served in parallel during a migration window.

---

## Response Envelope

### Success Response

All successful responses wrap data in a standard envelope:

```json
{
  "data": { ... },
  "meta": {
    "timestamp": "2026-03-30T14:22:00Z",
    "request_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

- `data` -- the response payload. Can be an object or an array.
- `meta.timestamp` -- ISO 8601 UTC timestamp of when the response was generated.
- `meta.request_id` -- UUID v4 that uniquely identifies the request for tracing and debugging.

For list endpoints, `meta` includes pagination info (see Pagination below).

### Error Response

All error responses use this format:

```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "No lines found for game ID abc-123",
    "details": {}
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:00Z",
    "request_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

- `error.code` -- machine-readable error code (uppercase snake_case).
- `error.message` -- human-readable description of the error.
- `error.details` -- optional object with additional context (e.g., validation errors per field).

### Common Error Codes

| Code                   | HTTP Status | Description                                                |
| ---------------------- | ----------- | ---------------------------------------------------------- |
| `VALIDATION_ERROR`     | 400         | Request body or query parameters failed validation         |
| `INVALID_PARAMETER`    | 400         | A specific parameter has an invalid value                  |
| `RESOURCE_NOT_FOUND`   | 404         | The requested resource does not exist                      |
| `DUPLICATE_RESOURCE`   | 409         | Resource already exists (idempotency check)                |
| `UNPROCESSABLE_ENTITY` | 422         | Request is syntactically valid but semantically wrong      |
| `INTERNAL_ERROR`       | 500         | Unexpected server error                                    |
| `SERVICE_UNAVAILABLE`  | 503         | Service is temporarily unavailable or a dependency is down |
| `DEPENDENCY_ERROR`     | 502         | An upstream service call failed                            |
| `TIMEOUT`              | 504         | An upstream service call timed out                         |

---

## HTTP Status Codes

| Status                      | Usage                                             |
| --------------------------- | ------------------------------------------------- |
| `200 OK`                    | Successful GET, PUT, or PATCH                     |
| `201 Created`               | Successful POST that creates a resource           |
| `204 No Content`            | Successful DELETE or action with no response body |
| `400 Bad Request`           | Malformed request, invalid parameters             |
| `404 Not Found`             | Resource does not exist                           |
| `409 Conflict`              | Duplicate resource (idempotency key collision)    |
| `422 Unprocessable Entity`  | Valid syntax but invalid semantics                |
| `500 Internal Server Error` | Unexpected failure                                |
| `502 Bad Gateway`           | Upstream dependency returned an error             |
| `503 Service Unavailable`   | Service or critical dependency is down            |
| `504 Gateway Timeout`       | Upstream dependency timed out                     |

---

## Pagination

List endpoints use **cursor-based pagination** to handle large result sets efficiently.

### Request Parameters

| Parameter | Type   | Default | Description                                                   |
| --------- | ------ | ------- | ------------------------------------------------------------- |
| `limit`   | int    | 50      | Maximum number of items to return (max 200)                   |
| `cursor`  | string | (none)  | Opaque cursor from a previous response to fetch the next page |

### Response Meta

When a list endpoint supports pagination, the `meta` object includes:

```json
{
  "data": [ ... ],
  "meta": {
    "timestamp": "2026-03-30T14:22:00Z",
    "request_id": "uuid",
    "pagination": {
      "limit": 50,
      "has_more": true,
      "next_cursor": "eyJpZCI6IjEyMyIsInRzIjoiMjAyNi0wMy0zMFQxNDoyMjowMFoifQ=="
    }
  }
}
```

- `pagination.limit` -- the limit that was applied.
- `pagination.has_more` -- whether there are more results beyond this page.
- `pagination.next_cursor` -- opaque cursor to pass as the `cursor` parameter for the next page. Only present when
  `has_more` is true.

Cursors are opaque base64-encoded strings. Clients must not parse or construct them.

---

## Filtering

List endpoints accept query parameters for filtering. Conventions:

| Pattern         | Example                                    | Description                               |
| --------------- | ------------------------------------------ | ----------------------------------------- |
| Exact match     | `?league=NBA`                              | Filter by exact value                     |
| Multiple values | `?league=NBA,NFL`                          | Comma-separated for OR matching           |
| Date range      | `?date_from=2026-03-01&date_to=2026-03-31` | Inclusive start and end dates (ISO 8601)  |
| Minimum/maximum | `?min_edge=3.0`                            | Numeric threshold filters                 |
| Boolean         | `?is_stale=false`                          | Boolean flags (`true`/`false` as strings) |
| Status          | `?status=FINAL`                            | Enum value filters                        |

All filter parameters are optional. When omitted, no filter is applied for that dimension.

---

## Data Serialization

- **Content-Type:** Always `application/json` for request and response bodies.
- **Date/time:** ISO 8601 with timezone: `2026-03-30T14:22:00Z`.
- **Odds:** American format as integers (e.g., `-110`, `+150`). Decimal odds included as floats for convenience.
- **Probabilities:** Floats with up to 4 decimal places (e.g., `0.5234`).
- **Monetary values:** Integers in cents where applicable, or floats in units for paper trading.
- **Null handling:** Absent fields are omitted from responses unless the field's absence is semantically meaningful.
- **Enums:** Uppercase strings matching the enum definitions in the domain model.

---

## Idempotency

State-changing POST endpoints that create resources support idempotency via the `X-Idempotency-Key` header:

```http
POST /api/v1/emulator/bets
X-Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

- The server checks if a resource with this idempotency key already exists.
- If so, the existing resource is returned with `200 OK` instead of creating a duplicate.
- Idempotency keys expire after 24 hours.
- Applicable endpoints: `POST /api/v1/emulator/bets`, `POST /api/v1/sim/simulations`, `POST
/api/v1/predict/predictions`.

---

## Authentication

Authentication is **deferred**. All services currently operate within an internal Docker network with no external
access. When external access or multi-user support is needed, shared API key or JWT validation middleware will be added
to each service.

For now, all endpoints are unauthenticated.

---

## Health Checks

Every service exposes a health endpoint at:

```text
GET /api/v1/{service}/health
```

Response:

```json
{
  "data": {
    "status": "healthy",
    "service": "lines-service",
    "version": "1.0.0",
    "uptime_seconds": 86400,
    "dependencies": {
      "database": "healthy",
      "redis": "healthy"
    }
  },
  "meta": {
    "timestamp": "2026-03-30T14:22:00Z",
    "request_id": "uuid"
  }
}
```

Status values: `healthy`, `degraded` (partially functional), `unhealthy` (critical failure).

---

## Timeouts

Callers set timeouts appropriate to the operation:

| Operation Type        | Timeout    |
| --------------------- | ---------- |
| Data lookups (GET)    | 5 seconds  |
| Simulation requests   | 5 minutes  |
| Prediction generation | 2 minutes  |
| LLM analysis requests | 30 seconds |
| Paper bet placement   | 5 seconds  |

---

## API Contract Files

| File                                                   | Service            | Framework        |
| ------------------------------------------------------ | ------------------ | ---------------- |
| [lines-service-api.md](lines-service-api.md)           | lines-service      | Go / Echo        |
| [statistics-service-api.md](statistics-service-api.md) | statistics-service | Go / Echo        |
| [simulation-engine-api.md](simulation-engine-api.md)   | simulation-engine  | Python / FastAPI |
| [prediction-engine-api.md](prediction-engine-api.md)   | prediction-engine  | Python / FastAPI |
| [bookie-emulator-api.md](bookie-emulator-api.md)       | bookie-emulator    | Python / FastAPI |
| [agent-api.md](agent-api.md)                           | agent              | Python / FastAPI |
