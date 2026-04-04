# Testing Strategy

Testing approach for BookieBreaker, a distributed sports prediction system spanning 11 repositories across Go, Python, and TypeScript.

---

## 1. Testing Philosophy

### Test Pyramid

BookieBreaker follows the standard test pyramid, weighted for a microservices architecture:

```
        /  E2E  \          Few — full pipeline tests against Docker Compose
       /----------\
      / Integration \      Moderate — service + real DB, service + recorded APIs
     /----------------\
    /    Unit Tests     \  Many — pure logic, no I/O, fast feedback
   /____________________\
```

**What to test at each level:**

| Level | Scope | Speed | What it validates |
|-------|-------|-------|-------------------|
| Unit | Single function or method | < 1ms each | Business logic, data transformations, calculations, edge cases |
| Integration | Service + real database or recorded API responses | < 5s each | Query correctness, schema compatibility, serialization, API response handling |
| Contract | Service API vs OpenAPI spec | < 2s each | Services conform to their published API contracts |
| End-to-End | Full pipeline across multiple services | < 60s each | Data flows correctly through the system, pipeline produces expected outputs |

**Guiding principles:**
- Unit tests are the primary defense. Every calculation, transformation, and decision branch gets a unit test.
- Integration tests prove the boundaries work. Database queries return what you expect. External API responses parse correctly.
- Contract tests catch drift between services. If lines-service changes its response shape, contract tests in consuming services fail.
- E2E tests are smoke tests, not exhaustive. They prove the pipeline runs, not that every edge case is handled.

---

## 2. Unit Tests per Language

### Go (lines-service, statistics-service, cli)

**Framework:** Standard library `testing` package + [testify](https://github.com/stretchr/testify) for assertions and mocks.

**Conventions:**
- Table-driven tests for functions with multiple input/output combinations
- Test files live alongside source: `handler.go` / `handler_test.go`
- Use `testify/assert` for assertions, `testify/require` for fatal assertions
- Use `testify/mock` for interface mocking

```go
// Example: table-driven test for odds conversion
func TestAmericanToDecimal(t *testing.T) {
    tests := []struct {
        name     string
        american int
        want     float64
    }{
        {"positive favorite", 150, 2.50},
        {"negative favorite", -200, 1.50},
        {"even money", 100, 2.00},
        {"heavy favorite", -1000, 1.10},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := AmericanToDecimal(tt.american)
            assert.InDelta(t, tt.want, got, 0.001)
        })
    }
}
```

**Run:** `go test ./...` from the repo root. Use `go test -race ./...` in CI to catch data races.

### Python (simulation-engine, prediction-engine, agent, mcp-server, bookie-emulator)

**Framework:** [pytest](https://docs.pytest.org/) + [pytest-asyncio](https://github.com/pytest-dev/pytest-asyncio) for async endpoint tests.

**Conventions:**
- Tests live in `tests/` directory at the repo root
- Use `conftest.py` for shared fixtures (database sessions, test clients, mock data)
- Use `pytest-cov` for coverage reporting
- Use `@pytest.mark.asyncio` for async tests

```python
# Example: simulation distribution test
import pytest
import numpy as np
from src.simulation.monte_carlo import simulate_game

class TestSimulateGame:
    def test_score_distributions_are_non_negative(self, nfl_team_params):
        result = simulate_game(nfl_team_params, iterations=1000)
        assert np.all(result.home_scores >= 0)
        assert np.all(result.away_scores >= 0)

    def test_spread_distribution_is_centered(self, nfl_team_params):
        result = simulate_game(nfl_team_params, iterations=10000)
        # Spread should be roughly centered for evenly matched teams
        assert abs(np.mean(result.spreads)) < 5.0

    @pytest.mark.parametrize("sport", ["NFL", "NBA", "MLB"])
    def test_score_ranges_per_sport(self, sport, team_params_factory):
        params = team_params_factory(sport)
        result = simulate_game(params, iterations=1000)
        # Sport-specific sanity checks
        if sport == "NFL":
            assert np.mean(result.home_scores) < 50
        elif sport == "NBA":
            assert np.mean(result.home_scores) > 70
```

**Run:** `uv run pytest --cov` from the repo root.

### TypeScript (ui)

**Framework:** [Vitest](https://vitest.dev/) for component/unit tests, [Playwright](https://playwright.dev/) for browser E2E tests.

**Conventions:**
- Component tests in `src/lib/components/__tests__/`
- Page tests colocated: `src/routes/edges/+page.test.ts`
- Playwright tests in `tests/e2e/`

```typescript
// Example: Vitest component test
import { render, screen } from '@testing-library/svelte';
import { describe, it, expect } from 'vitest';
import EdgeCard from '$lib/components/EdgeCard.svelte';

describe('EdgeCard', () => {
  it('displays positive EV in green', () => {
    render(EdgeCard, { props: { edge: { ev: 5.2, market: 'SPREAD' } } });
    const evElement = screen.getByText('+5.2%');
    expect(evElement).toHaveClass('text-green-500');
  });

  it('displays negative EV in red', () => {
    render(EdgeCard, { props: { edge: { ev: -2.1, market: 'TOTAL' } } });
    const evElement = screen.getByText('-2.1%');
    expect(evElement).toHaveClass('text-red-500');
  });
});
```

**Run:** `pnpm test` (Vitest) and `pnpm test:e2e` (Playwright).

---

## 3. Integration Tests

### Service + Real Database

Use [testcontainers](https://testcontainers.com/) to spin up ephemeral Postgres (with TimescaleDB) and Redis containers for integration tests. Each test gets a clean database.

**Go (testcontainers-go):**

```go
func TestLineSnapshotRepository(t *testing.T) {
    ctx := context.Background()
    // Start a real TimescaleDB container
    pgContainer, err := postgres.Run(ctx,
        "timescale/timescaledb:latest-pg16",
        postgres.WithDatabase("test"),
    )
    require.NoError(t, err)
    defer pgContainer.Terminate(ctx)

    connStr, _ := pgContainer.ConnectionString(ctx)
    db := connectAndMigrate(connStr)

    repo := NewLineSnapshotRepository(db)

    // Test real queries against real Postgres
    snapshot := buildTestSnapshot()
    err = repo.Insert(ctx, snapshot)
    require.NoError(t, err)

    result, err := repo.GetByGame(ctx, snapshot.GameExternalID)
    require.NoError(t, err)
    assert.Len(t, result, 1)
}
```

**Python (testcontainers-python):**

```python
import pytest
from testcontainers.postgres import PostgresContainer
from testcontainers.redis import RedisContainer

@pytest.fixture(scope="session")
def postgres_url():
    with PostgresContainer("timescale/timescaledb:latest-pg16") as pg:
        yield pg.get_connection_url()

@pytest.fixture(scope="session")
def redis_url():
    with RedisContainer("redis:7-alpine") as r:
        yield f"redis://{r.get_container_host_ip()}:{r.get_exposed_port(6379)}"
```

### Service + External API (VCR/Cassette Pattern)

Record real API responses once, replay them in tests. This avoids hitting rate-limited external APIs in CI while still testing real response parsing.

**Go:** Use [go-vcr](https://github.com/dnaeon/go-vcr) or [httpmock](https://github.com/jarcoal/httpmock) with recorded fixtures.

**Python:** Use [vcrpy](https://github.com/kevin1024/vcrpy) or [pytest-recording](https://github.com/kiwicom/pytest-recording):

```python
@pytest.mark.vcr()
def test_fetch_nfl_schedule(statistics_client):
    """Uses recorded cassette from tests/cassettes/test_fetch_nfl_schedule.yaml"""
    schedule = statistics_client.get_schedule("NFL", season=2025)
    assert len(schedule.games) > 200
    assert schedule.games[0].home_team is not None
```

Cassette files are committed to the repo under `tests/cassettes/`. Re-record periodically when external API response shapes change by deleting the cassette and running the test with network access.

---

## 4. Contract Tests

### OpenAPI Spec Validation

Every service publishes an OpenAPI spec. Contract tests verify that each service's actual responses conform to its spec.

**Go services (compile-time contracts):**

Go services use [oapi-codegen](https://github.com/oapi-codegen/oapi-codegen) to generate server interfaces and request/response types directly from OpenAPI specs. The Go compiler enforces contract compliance: if a handler returns a struct that doesn't match the generated type, it won't compile. This gives contract validation at build time with zero test overhead.

**Python services (schemathesis):**

Use [schemathesis](https://github.com/schemathesis/schemathesis) to auto-generate contract tests from FastAPI's auto-generated OpenAPI spec:

```bash
# Run against a live service (local or Docker)
uv run schemathesis run http://localhost:8003/openapi.json --checks all

# Or as a pytest plugin
uv run pytest --schemathesis http://localhost:8003/openapi.json
```

Schemathesis automatically:
- Generates random valid payloads for every endpoint
- Verifies response status codes match the spec
- Validates response body schemas
- Detects undocumented response codes

**Cross-service contract validation:**

Each consuming service should validate that the upstream service's OpenAPI spec hasn't changed in a breaking way. Store a snapshot of each upstream spec and diff against the live version in CI:

```bash
# In lines-service CI: verify statistics-service spec hasn't broken us
diff <(curl -s http://statistics-service:8002/openapi.json) ./api/upstream/statistics-service.json
```

---

## 5. End-to-End Pipeline Tests

### Full Pipeline Test

An E2E test exercises the complete prediction pipeline:

```
lines ingested → stats fetched → simulation run → prediction calibrated → edge detected → paper bet placed
```

**Setup:** Run against the full Docker Compose stack with seeded data.

```python
# tests/e2e/test_prediction_pipeline.py

import httpx
import pytest

AGENT_URL = "http://localhost:8006"
LINES_URL = "http://localhost:8001"
EMULATOR_URL = "http://localhost:8005"

@pytest.mark.e2e
class TestPredictionPipeline:
    def test_full_pipeline_produces_predictions(self):
        """Trigger pipeline and verify predictions are generated."""
        # 1. Verify lines exist (from seed data)
        lines = httpx.get(f"{LINES_URL}/api/v1/lines/current?league=NFL").json()
        assert len(lines["data"]) > 0
        game_id = lines["data"][0]["game_external_id"]

        # 2. Trigger pipeline via agent
        response = httpx.post(
            f"{AGENT_URL}/api/v1/pipeline/run",
            json={"game_ids": [game_id]},
            timeout=120.0,  # Simulations take time
        )
        assert response.status_code == 200
        result = response.json()

        # 3. Verify predictions were generated
        assert len(result["data"]["predictions"]) > 0
        prediction = result["data"]["predictions"][0]
        assert 0.0 <= prediction["probability"] <= 1.0
        assert prediction["confidence_interval"] is not None

    def test_edge_detection_places_paper_bet(self):
        """When an edge exceeds threshold, a paper bet should be placed."""
        # Trigger pipeline with a game that has known edge in seed data
        response = httpx.post(
            f"{AGENT_URL}/api/v1/pipeline/run",
            json={"game_ids": ["SEED_GAME_WITH_EDGE"]},
            timeout=120.0,
        )
        assert response.status_code == 200

        # Check bookie-emulator for the paper bet
        bets = httpx.get(f"{EMULATOR_URL}/api/v1/bets?status=OPEN").json()
        assert len(bets["data"]) > 0
```

**Run E2E tests:**

```bash
# Start the full stack
task up
task db:migrate
task db:seed

# Run E2E tests
cd bookie-breaker-infra-ops && uv run pytest tests/e2e/ -m e2e --timeout=180
```

---

## 6. Simulation Validation

Simulation correctness is critical because all downstream predictions depend on it. These tests are domain-specific and statistical in nature.

### Distribution Sanity Tests

```python
class TestSimulationDistributions:
    def test_nfl_scores_follow_expected_distribution(self, nfl_params):
        result = simulate_game(nfl_params, iterations=50000)
        mean_total = np.mean(result.home_scores + result.away_scores)
        # NFL games average 40-50 total points
        assert 35 < mean_total < 55

    def test_nba_scores_follow_expected_distribution(self, nba_params):
        result = simulate_game(nba_params, iterations=50000)
        mean_total = np.mean(result.home_scores + result.away_scores)
        # NBA games average 210-230 total points
        assert 190 < mean_total < 250
```

### Calibration Tests

Verify that simulation probabilities are well-calibrated against historical game results:

```python
def test_simulation_calibration(historical_games, team_params_lookup):
    """
    For games where simulation says 70% home win,
    home teams should win roughly 70% of the time.
    """
    buckets = defaultdict(list)  # probability_bucket -> [actual_outcome]

    for game in historical_games:
        result = simulate_game(
            team_params_lookup[game.home_team, game.away_team],
            iterations=10000,
        )
        prob = result.home_win_probability
        bucket = round(prob, 1)  # 0.0, 0.1, ..., 1.0
        buckets[bucket].append(1 if game.home_won else 0)

    for bucket_prob, outcomes in buckets.items():
        if len(outcomes) < 20:
            continue  # Skip thin buckets
        actual_rate = np.mean(outcomes)
        # Calibration: predicted probability should be close to actual win rate
        assert abs(actual_rate - bucket_prob) < 0.10, (
            f"Bucket {bucket_prob}: predicted {bucket_prob:.0%}, actual {actual_rate:.0%}"
        )
```

### Convergence Tests

Verify that simulation results stabilize as iteration count increases:

```python
def test_simulation_converges(nfl_params):
    """Results at 50k iterations should be close to results at 10k."""
    result_10k = simulate_game(nfl_params, iterations=10000)
    result_50k = simulate_game(nfl_params, iterations=50000)

    # Mean spread should converge within 0.5 points
    assert abs(np.mean(result_10k.spreads) - np.mean(result_50k.spreads)) < 0.5
    # Win probability should converge within 2%
    assert abs(result_10k.home_win_probability - result_50k.home_win_probability) < 0.02
```

---

## 7. Model Evaluation

### Walk-Forward Backtesting

Models are evaluated on held-out temporal data to prevent look-ahead bias:

```
Training: weeks 1-10 → Evaluate: week 11
Training: weeks 1-11 → Evaluate: week 12
Training: weeks 1-12 → Evaluate: week 13
...
```

Never use future data to predict past games. The walk-forward approach mimics real-world deployment where the model only has access to data available at prediction time.

### Evaluation Metrics

| Metric | What it measures | Target |
|--------|-----------------|--------|
| Brier score | Mean squared prediction error (lower is better) | < 0.25 for game outcomes |
| Log loss | Penalizes confident wrong predictions (lower is better) | < 0.69 (better than coin flip) |
| Calibration error (ECE) | Gap between predicted and actual probabilities | < 0.05 |
| ROI (paper trading) | Return on investment at unit stakes | > 0% sustained over 500+ bets |
| CLV (Closing Line Value) | Were predictions better than the closing line? | Positive mean CLV |

### A/B Model Comparison

When training a new model version:

1. Run walk-forward backtest on the same time period with both old and new models
2. Compare Brier score, log loss, and calibration error
3. Run a paired statistical test (Wilcoxon signed-rank) to check significance
4. Only deploy if the new model is statistically better or equivalent with other advantages (speed, simplicity)

```python
def compare_models(model_a_preds, model_b_preds, actuals):
    brier_a = brier_score_loss(actuals, model_a_preds)
    brier_b = brier_score_loss(actuals, model_b_preds)
    _, p_value = wilcoxon(
        (model_a_preds - actuals) ** 2,
        (model_b_preds - actuals) ** 2,
    )
    return {
        "brier_a": brier_a,
        "brier_b": brier_b,
        "improvement": brier_a - brier_b,
        "p_value": p_value,
        "significant": p_value < 0.05,
    }
```

---

## 8. Test Data Management

### Fixtures vs Factories vs Recorded Data

| Approach | When to use | Examples |
|----------|-------------|---------|
| **Static fixtures** (JSON/SQL files) | Stable reference data that rarely changes | Sportsbook definitions, league configurations, sport-specific constants |
| **Factories** (programmatic generation) | Tests needing many variations of the same entity | Random game generators, prediction factories with configurable confidence ranges |
| **Recorded data** (VCR cassettes) | External API response testing | Odds API responses, nfl_data_py output, nba_api game logs |
| **Seed data** (curated SQL scripts) | Full-stack development and E2E tests | See seed data strategy in dev-workflow.md |

### Generating Realistic Test Data

Each sport has different statistical profiles. Factories should produce data within realistic ranges:

```python
# tests/factories.py
import factory
from faker import Faker

class NFLGameFactory(factory.Factory):
    class Meta:
        model = Game

    league = "NFL"
    home_score = factory.LazyFunction(lambda: random.randint(0, 56))
    away_score = factory.LazyFunction(lambda: random.randint(0, 56))
    home_team = factory.LazyFunction(lambda: random.choice(NFL_TEAMS))
    away_team = factory.LazyFunction(lambda: random.choice(NFL_TEAMS))

class NBAGameFactory(factory.Factory):
    class Meta:
        model = Game

    league = "NBA"
    home_score = factory.LazyFunction(lambda: random.randint(80, 140))
    away_score = factory.LazyFunction(lambda: random.randint(80, 140))
```

### Sensitive Data Handling

- **No real API keys in tests.** Use placeholder values (`test-api-key-xxx`) or environment variables that CI injects.
- **No real user data.** All test data is synthetic.
- **Cassette scrubbing.** VCR cassettes are recorded with API keys redacted via filter configuration:

```python
# conftest.py
@pytest.fixture(scope="module")
def vcr_config():
    return {
        "filter_headers": ["authorization", "x-api-key"],
        "filter_query_parameters": ["apiKey"],
    }
```

---

## 9. CI Integration

### Per-PR Tests (every push, must pass to merge)

These run on every pull request and block merging if they fail. Target: under 5 minutes total.

| Test type | What runs | Time budget |
|-----------|-----------|-------------|
| Unit tests | All unit tests across the changed repo(s) | < 2 min |
| Lint + format | `golangci-lint`, `ruff check`, `ruff format --check`, `eslint`, `prettier` | < 1 min |
| Type checking | `go build`, `mypy --strict`, `svelte-check` | < 1 min |
| Contract tests (Go) | Compile-time via oapi-codegen (implicit in `go build`) | 0 sec |
| Contract tests (Python) | `schemathesis` against the service's own OpenAPI spec | < 1 min |

### Nightly Tests (scheduled, longer-running)

These run on a schedule (e.g., 2 AM UTC) and alert on failure. They are too slow or resource-intensive for every PR.

| Test type | What runs | Time budget |
|-----------|-----------|-------------|
| Integration tests | Testcontainers-based tests with real Postgres and Redis | < 10 min |
| E2E pipeline tests | Full pipeline against Docker Compose stack | < 15 min |
| Simulation validation | Distribution, calibration, and convergence tests | < 20 min |
| Model evaluation | Walk-forward backtest on latest data | < 30 min |
| Cross-service contracts | Verify all services against each other's OpenAPI specs | < 5 min |
| Schemathesis fuzzing | Extended property-based testing against all API endpoints | < 15 min |

### Test Parallelization

| Language | Strategy |
|----------|----------|
| Go | `go test -parallel $(nproc) ./...` -- Go's test runner parallelizes by default. Use `t.Parallel()` in individual tests. |
| Python | `pytest -n auto` (via `pytest-xdist`) -- distributes tests across CPU cores. Integration tests with testcontainers run serially to avoid port conflicts. |
| TypeScript | Vitest runs in parallel by default. Playwright shards across workers: `playwright test --shard=1/4`. |

### CI Pipeline Structure (GitHub Actions)

```yaml
# Per-repo: .github/workflows/ci.yml
name: CI
on: [pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: golangci-lint run  # or: uv run ruff check . / pnpm lint

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: go test -race ./...  # or: uv run pytest --cov / pnpm test

  # Nightly (only in infra-ops repo)
  # .github/workflows/nightly.yml
  # Triggers: schedule: - cron: '0 2 * * *'
  # Runs: integration, e2e, simulation validation, model evaluation
```

### Failure Notifications

- **PR tests:** GitHub PR status checks block merge. Failures are visible in the PR.
- **Nightly tests:** Send failure notifications via GitHub Actions notification (email or Slack webhook). Nightly failures indicate regressions, external API changes, or model drift.
