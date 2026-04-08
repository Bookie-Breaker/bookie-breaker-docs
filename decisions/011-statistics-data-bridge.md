# ADR-011: Statistics Service Data Bridge Strategy

## Status

Accepted

## Context

The statistics-service is written in Go (ADR-010) for fast HTTP serving and Redis caching, but the recommended data
sources (ADR-008) are a mix of:

- **Direct REST APIs** (Go-callable): NBA.com undocumented endpoints, CFBD API, CBBD API
- **Python data packages** (not Go-callable): nfl_data_py (downloads pre-processed parquet/CSV files), pybaseball
  (scrapes Baseball Savant, Baseball Reference, FanGraphs HTML), baseballr (R package, scrapes stats.ncaa.org)

Phase 1 targets NBA only, where Go can call NBA.com endpoints directly. Phase 6 adds NFL, MLB, NCAA Football, NCAA
Basketball, and NCAA Baseball — three of which require Python or R packages.

Options considered:

1. **Rewrite statistics-service in Python** — Eliminates the bridge problem but sacrifices Go's HTTP performance
   advantage.
2. **Go-native for all sports** — Would require reverse-engineering nflverse data pipelines and HTML scraping in Go.
   Significant work and fragile.
3. **Go service + Python sidecar at Phase 6** — Keeps Go for serving/caching, adds Python for data fetching when needed.
4. **Commercial API (Sportradar/SportsDataIO)** — Proper REST APIs for all sports, but $500+/month per sport.

## Decision

Keep statistics-service in Go. For Phase 1 through Phase 5, Go calls data source APIs directly:

- **NBA:** Call `stats.nba.com` endpoints directly from Go (same endpoints nba_api wraps)
- **NCAA Football:** Call CFBD REST API from Go
- **NCAA Basketball:** Call CBBD REST API from Go

At Phase 6, add a **Python sidecar container** for sports that require Python packages:

- **NFL:** nfl_data_py (downloads nflverse parquet/CSV files)
- **MLB:** pybaseball (scrapes Baseball Savant + FanGraphs)
- **NCAA Baseball:** baseballr NCAA functions (via sportsdataverse-py or subprocess)

The Go statistics-service calls the Python sidecar via internal HTTP, caches all results in Redis, and serves the
unified REST API to consumers.

## Consequences

### Positive

- Go statistics-service delivers fast HTTP serving and efficient Redis caching for all consumers
- No Python dependency needed until Phase 6 — simpler stack for the first 5 phases
- NBA, CFBD, and CBBD work natively in Go with no bridge required
- Python sidecar is a thin data-fetching layer, not a full service — minimal operational overhead

### Negative

- At Phase 6, statistics-service becomes two containers (Go + Python sidecar) for one logical service
- NBA.com endpoints are undocumented and can break seasonally; nba_api's Python community catches these changes faster
  than we will in Go
- Python sidecar requires its own Dockerfile, health check, and dependency management

### Neutral

- The sidecar pattern is well-established in microservice architectures
- If NBA.com endpoint instability becomes a problem pre-Phase 6, the Python sidecar can be introduced earlier
- If budget allows, Sportradar/SportsDataIO could replace the sidecar with a proper REST API at any time
