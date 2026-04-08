# ADR-007: Lines/Odds Data Source Selection

## Status

Accepted

## Context

The lines-service needs real-time and historical betting lines/odds data across 6 leagues (NFL, NBA, MLB, NCAA Football, NCAA Basketball, NCAA Baseball) and all bet types (spreads, totals, moneylines, props, futures, live/in-game). After evaluating 12+ providers (see `research/lines-data-sources.md`), we needed to select a primary source, live streaming supplement, and fallback.

Key constraints:

- NCAA Baseball coverage is extremely limited across the industry
- Enterprise providers (Sportradar, TxODDS, Unabated) cost $2,000-3,000+/month — disproportionate for this project
- No sportsbook (DraftKings, FanDuel, BetMGM) offers a public API — all access is through aggregators
- Pinnacle's public API shut down July 2025

## Decision

A three-tier data source strategy:

1. **Primary: The Odds API** (~$59-249/month)
   - Covers all 6 target leagues including NCAA Baseball (confirmed)
   - 40+ sportsbooks aggregated
   - Simple REST API, well-documented
   - 99.9% SLA
   - Sufficient for development and initial production

2. **Live/Streaming Supplement: SharpAPI** (~$79-229/month)
   - SSE streaming at sub-89ms latency
   - Built-in +EV detection capabilities
   - Covers major leagues; NCAA coverage varies

3. **Deep Coverage Fallback: OddsJam** (~$500+/month)
   - Deepest prop market coverage
   - Historical odds data for validation
   - Confirmed NCAA Baseball coverage
   - Only add if The Odds API coverage proves insufficient

**NCAA Baseball fallback:** If aggregator coverage is inadequate, build a lightweight seasonal scraper for DraftKings/FanDuel NCAA Baseball lines (February-June only). Accept that prop and futures markets will be thin or absent for NCAA Baseball.

**Development phase:** Use The Odds API free tier and SharpAPI free tier for development. Production costs start at ~$138-478/month.

## Consequences

### Positive

- Low barrier to start — free tiers available for both primary and streaming sources
- The Odds API is the most documented and stable provider in the space
- Three-tier approach provides redundancy without upfront cost commitment
- SharpAPI's built-in +EV detection can serve as a validation signal

### Negative

- NCAA Baseball remains the weakest link regardless of provider choice
- The Odds API's 40+ sportsbooks is fewer than some competitors (Odds-API.io claims 265+)
- Adding OddsJam later significantly increases costs
- SharpAPI is a newer provider with less track record

### Neutral

- All three providers may need a normalization layer to standardize data formats
- Live/in-game betting latency requirements may demand upgrading to higher tiers over time
