# ADR-008: Statistics Data Source Selection

## Status
Accepted

## Context
The statistics-service needs historical and current sports statistics to feed the simulation engine (Monte Carlo) and prediction engine (ML models). After evaluating 12+ providers (see `research/statistics-data-sources.md`), we needed a primary + fallback source for each of the 6 target leagues.

Key findings:
- Professional leagues have excellent free/open-source Python ecosystems
- College data is significantly more fragmented and lower quality
- Sports Reference explicitly prohibits automated/ML use in their ToS
- Commercial APIs (Sportradar, SportsDataIO) are $500+/month per sport — not justified when free alternatives exist for most leagues
- NCAA Baseball has the worst data availability of all 6 leagues

## Decision
Use free, community-maintained Python packages as primary sources wherever possible. Reserve paid APIs as fallbacks only.

| League | Primary Source | Cost | Fallback |
|--------|---------------|------|----------|
| **NFL** | nfl_data_py | Free | ESPN endpoints |
| **NBA** | nba_api | Free | hoopR (via rpy2) |
| **MLB** | pybaseball (Statcast) | Free | baseballr (via rpy2) |
| **NCAA Football** | CFBD API | Free / $10/mo | cfbfastR |
| **NCAA Basketball** | CBBD API | Free tier | hoopR / ESPN endpoints |
| **NCAA Baseball** | baseballr NCAA functions | Free | ncaa_bbStats / collegebaseball Python packages |

**Data types prioritized:**
- Play-by-play data (essential for simulation engine)
- Box scores and game results (essential for model training/evaluation)
- Player-level stats and advanced metrics (essential for feature engineering)
- Team aggregates and rankings (useful for contextual features)

**Total estimated cost:** $0-10/month for all statistics data.

**NCAA Baseball mitigation:** Accept lower data quality. Use baseballr's NCAA functions (scrapes stats.ncaa.org via R, callable through rpy2 or subprocess). Supplement with collegebaseball Python package. Expect shallower historical depth and slower updates compared to pro leagues.

## Consequences

### Positive
- Near-zero data cost for all 6 leagues
- Python-native packages (nfl_data_py, nba_api, pybaseball) integrate directly with the ML/simulation pipeline
- Community-maintained packages benefit from ongoing improvements and bug fixes
- CFBD API is purpose-built for college football analytics — excellent quality for its niche

### Negative
- Free/unofficial APIs can break without notice (especially nba_api, ESPN endpoints)
- NCAA Baseball data quality is genuinely poor — models for this league will be less accurate
- Some R packages (baseballr, hoopR) require rpy2 bridge or subprocess calls, adding complexity
- No SLA guarantees on any free source — need resilient ingestion with caching

### Neutral
- Sports Reference / Stathead is off-limits for automated pipelines (ToS prohibits ML/commercial use) — useful only for manual validation
- ESPN endpoints are undocumented and may change — suitable as fallback only, not primary
- May need SportsDataIO or Sportradar later if free sources become unreliable at scale
