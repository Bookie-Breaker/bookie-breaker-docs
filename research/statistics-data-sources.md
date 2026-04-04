# Sports Statistics & Historical Data Sources Research

**Document Status:** Research Complete  
**Last Updated:** 2026-03-30  
**Author:** BookieBreaker Research Team  
**Purpose:** Evaluate data providers for Monte Carlo simulation and ML prediction models across NFL, NBA, MLB, NCAA Football, NCAA Basketball, and NCAA Baseball.

---

## 1. Executive Summary

### Key Findings

1. **Professional leagues have excellent free/open-source data ecosystems.** NFL (nfl_data_py / nflverse), NBA (nba_api), and MLB (pybaseball / Statcast) each have mature, community-maintained Python packages that provide play-by-play data, advanced metrics, and deep historical coverage at zero cost. These should serve as primary sources for their respective leagues.

2. **College data is fragmented and shallower.** NCAA data quality drops significantly compared to pro leagues. College football is best served by the CFBD API; college basketball has a newer but promising equivalent in CollegeBasketballData.com (CBBD). College baseball is the weakest link, relying on scrapers against stats.ncaa.org with limited historical depth and no Statcast-equivalent tracking data.

3. **Commercial APIs (Sportradar, SportsDataIO) are expensive but comprehensive.** Starting at ~$500/month per sport, they provide normalized, production-grade feeds across all leagues. They are best reserved as fallback or for real-time in-game data rather than historical bulk analysis.

4. **ESPN unofficial endpoints are useful supplements but carry risk.** They cover all six target leagues with no authentication required, but are undocumented, can change without notice, and are unsuitable as a sole primary source.

5. **Sports Reference is off-limits for automated access.** Their terms of use explicitly prohibit scraping, bot access, and use of their data to train AI/ML models. While their data quality is exceptional, they cannot be used as a programmatic source for BookieBreaker.

6. **The SportsDataverse ecosystem (cfbfastR, hoopR, baseballr) provides excellent R-based alternatives** that can be called from Python via rpy2 if needed, and the sportsdataverse-py package offers some Python-native equivalents.

### Cost Summary

| Source | Cost | Best For |
|--------|------|----------|
| nfl_data_py | Free | NFL primary |
| nba_api | Free | NBA primary |
| pybaseball | Free | MLB primary |
| CFBD API | Free (1K calls/mo) / $10/mo (75K calls) | NCAA Football primary |
| CBBD API | Free tier available | NCAA Basketball primary |
| ESPN Endpoints | Free (unofficial) | Cross-league supplement |
| baseballr (NCAA functions) | Free | NCAA Baseball primary |
| SportsDataIO | ~$500+/mo per sport | Enterprise fallback |
| Sportradar | Custom pricing (enterprise) | Enterprise fallback |

---

## 2. Detailed Provider Profiles

### 2.1 nfl_data_py / nflverse

| Attribute | Details |
|-----------|---------|
| **Sports/Leagues** | NFL only |
| **Data Types** | Play-by-play, weekly player stats, seasonal stats, rosters, win totals, scoring lines, officials, draft picks, draft pick values, schedules, team info, combine results, player ID mappings |
| **Advanced Metrics** | Expected points added (EPA), win probability (WP), completion probability (CP), expected yards after catch (xYAC); FTN charting data (2022+) |
| **Historical Depth** | Play-by-play from 1999 to present (~27 seasons); draft data goes back further |
| **Update Frequency** | Nightly during season; FTN charting within 48 hours of game completion |
| **Access Method** | Python package (`pip install nfl-data-py`); also `nflreadpy` as a newer port of R's nflreadr |
| **Pricing** | Free and open-source |
| **Data Quality** | Excellent. Gold standard for NFL analytics. Community-maintained with active contributors. |
| **ToS/Commercial Use** | Open-source (MIT license). Data sourced from nflfastR, nfldata, dynastyprocess, and Draft Scout. FTN charting data subset is provided with permission. |
| **Caching** | Local caching supported; reduces repeated download times by ~30% via optional float32 downcast |

**Strengths:** Deepest free NFL dataset available. EPA and WP models are industry-standard. Active community (nflverse GitHub org).  
**Weaknesses:** NFL only. FTN charting data limited to 2022+. No real-time/live data feed.

---

### 2.2 nba_api (stats.nba.com wrapper)

| Attribute | Details |
|-----------|---------|
| **Sports/Leagues** | NBA, WNBA, G-League |
| **Data Types** | Player stats, team stats, game logs, box scores, play-by-play, shot charts/locations, lineups, standings, draft history, synergy play types |
| **Advanced Metrics** | Tracking data (speed, distance, touches), hustle stats, matchup data, defensive metrics |
| **Historical Depth** | Basic stats back to 1946-47 season; play-by-play and advanced tracking from ~2013-14 forward [unverified exact start year for tracking] |
| **Update Frequency** | Near real-time during games (stats.nba.com updates live) |
| **Access Method** | Python package (`pip install nba_api`); current version 1.11.3 (Feb 2026) |
| **Pricing** | Free (wraps public NBA.com endpoints) |
| **Data Quality** | Very good for official stats. Tracking data is NBA-proprietary (Second Spectrum). Some endpoints deprecated each season (e.g., ScoreboardV2 deprecated for 2025-26, replaced by V3). |
| **ToS/Commercial Use** | Unofficial wrapper of NBA.com's undocumented APIs. NBA.com does not officially sanction this usage. Commercial use is a gray area -- the data is publicly accessible but not officially licensed for redistribution. |

**Strengths:** Enormous breadth of endpoints. Includes tracking data not available elsewhere for free. Active maintenance with version upgrades tracking NBA.com changes.  
**Weaknesses:** Endpoints break seasonally as NBA.com updates their internal API. Rate limiting can be aggressive. No official support channel. Requires careful error handling for deprecated endpoints.

---

### 2.3 pybaseball / Baseball Savant / Statcast

| Attribute | Details |
|-----------|---------|
| **Sports/Leagues** | MLB only (pro level) |
| **Data Types** | Pitch-level Statcast data (velocity, spin rate, exit velocity, launch angle, pitch coordinates), batting stats, pitching stats, fielding stats, standings, awards, player lookups |
| **Advanced Metrics** | xBA, xSLG, xwOBA, barrel rate, hard-hit rate, sprint speed, arm strength, perceived velocity, active spin (2020+) |
| **Historical Depth** | Statcast pitch-level data from 2008 (system introduction); launch speed/angle from 2015+; active spin from 2020+. Baseball Reference/FanGraphs stats go back to early 1900s. |
| **Update Frequency** | Statcast data available next day; season stats updated daily |
| **Access Method** | Python package (`pip install pybaseball`); scrapes Baseball Savant, Baseball Reference, and FanGraphs |
| **Pricing** | Free and open-source |
| **Data Quality** | Excellent. Statcast is the gold standard for baseball analytics. 700K+ pitches per season with rich metadata. Data subject to retroactive corrections. |
| **ToS/Commercial Use** | pybaseball is open-source. However, it scrapes from Baseball Savant (MLB), Baseball Reference (Sports Reference -- see restrictions in 2.5), and FanGraphs. Baseball Savant data is MLB-owned. Commercial redistribution of scraped data may violate source site ToS. |
| **Rate Limits** | Baseball Savant limits queries to 30,000 rows; pybaseball auto-chunks large date ranges into smaller requests |

**Strengths:** Unmatched pitch-level granularity. Combines three major data sources into one interface. Essential for any serious baseball modeling.  
**Weaknesses:** Scraping-based, so subject to source site changes. Large queries are slow. No real-time/live data.

---

### 2.4 collegefootballdata.com (CFBD) API

| Attribute | Details |
|-----------|---------|
| **Sports/Leagues** | NCAA Football (FBS and FCS) |
| **Data Types** | Play-by-play, box scores, team stats, player stats, game results, recruiting rankings, team talent composites, SP+ ratings, pregame win probabilities, schedules, venues, conferences, coaches |
| **Advanced Metrics** | PPA (predicted points added), success rates, havoc rates, SP+ [unverified if SP+ is still included in v2], EPA |
| **Historical Depth** | Play-by-play from ~2004; game results and basic stats go back further [unverified exact year] |
| **Update Frequency** | During season, updated within hours of game completion |
| **Access Method** | REST API (v2 as of May 2025); Python client (`pip install cfbd`); R client via cfbfastR |
| **Pricing** | Free tier: 1,000 API calls/month. Patreon Tier 3 ($10/month): 75,000 calls/month + GraphQL API with real-time subscriptions. Custom tiers available for higher volume. |
| **Data Quality** | Very good. The most comprehensive free college football data source. Community-driven with active development. |
| **ToS/Commercial Use** | API key required. Free tier sufficient for development/research. Commercial use likely requires Patreon tier or direct arrangement. |

**Strengths:** Best-in-class for college football. Rich advanced metrics. Active community and maintainer. Affordable paid tier.  
**Weaknesses:** College football only. API v1 was shut down in 2025 -- must use v2. Free tier (1K calls/month) may be insufficient for bulk historical pulls.

---

### 2.5 Sports Reference / Stathead

| Attribute | Details |
|-----------|---------|
| **Sports/Leagues** | NFL (Pro Football Reference), NBA (Basketball Reference), MLB (Baseball Reference), NCAA Football, NCAA Basketball, college baseball (via minor league/college sections) |
| **Data Types** | Comprehensive box scores, seasonal stats, career stats, game logs, splits, advanced stats, historical records |
| **Advanced Metrics** | PFR: ANY/A, DVOA-adjacent stats. BBRef: BPM, VORP, WS. BRef: WAR, OPS+, ERA+ |
| **Historical Depth** | Exceptional. MLB back to 1876. NBA back to 1946. NFL back to 1920. |
| **Update Frequency** | Daily during seasons |
| **Access Method** | Web-only. No official API. Stathead subscription ($9/month) for advanced queries via web UI. |
| **Pricing** | Stathead: $9/month (50% student discount, 25% military/teacher/medical). No API product. |
| **Data Quality** | Gold standard for historical sports data accuracy. Widely cited in academic research. |
| **ToS/Commercial Use** | **CRITICAL RESTRICTION:** ToS explicitly prohibit automated access (scripts, bots, scrapers). Rate limit: 20 requests/minute (10 for FBref/Stathead). Explicitly prohibit use of data to train AI/ML models without permission. Some datasets preclude any redistribution. |

**Strengths:** Deepest historical coverage. Highest data accuracy reputation. Comprehensive advanced metrics.  
**Weaknesses:** **Cannot be used programmatically for BookieBreaker.** No API. Aggressive anti-bot measures. ToS prohibit AI/ML training use. Manual Stathead queries only.

**Recommendation:** Do not use as a data source for BookieBreaker's automated pipeline. Useful only for manual validation and spot-checking.

---

### 2.6 ESPN API (Unofficial Endpoints)

| Attribute | Details |
|-----------|---------|
| **Sports/Leagues** | NFL, NBA, MLB, NHL, WNBA, NCAA Football, NCAA Men's/Women's Basketball, NCAA Baseball, MLS, international soccer, 20+ sports total |
| **Data Types** | Scores/scoreboard, schedules, standings, team rosters, news headlines, some box score data |
| **Advanced Metrics** | Limited. Primarily basic stats and scores. Some QBR data for football. |
| **Historical Depth** | Varies by endpoint; generally current season + a few prior seasons. Not designed for deep historical pulls. |
| **Update Frequency** | Near real-time for scores; varies for other data |
| **Access Method** | HTTP GET requests to `site.api.espn.com` endpoints. No authentication required. Community-documented on GitHub (pseudo-r/Public-ESPN-API). |
| **Pricing** | Free |
| **Data Quality** | Good for scores and schedules. Inconsistent for detailed stats. Data structure can change without warning. |
| **ToS/Commercial Use** | **Unofficial and undocumented.** ESPN can modify or remove endpoints at any time. No guaranteed availability. Commercial use is not sanctioned. |

**Strengths:** Covers all six BookieBreaker leagues. No auth needed. Good for scores, schedules, and basic game info.  
**Weaknesses:** Shallow stats depth. No play-by-play. Unreliable long-term. Not suitable as a primary analytical data source.

---

### 2.7 NCAA Official Stats Portal (stats.ncaa.org)

| Attribute | Details |
|-----------|---------|
| **Sports/Leagues** | All NCAA sports across D-I, D-II, D-III: Football, Men's/Women's Basketball, Baseball, Softball, Hockey, Lacrosse, Soccer, and more |
| **Data Types** | Team and individual leaders, game results, schedules, basic box scores |
| **Advanced Metrics** | Minimal. Basic counting stats and rate stats only. |
| **Historical Depth** | Varies by sport. Generally several years of data available. |
| **Update Frequency** | Updated during seasons, generally within 24 hours of games |
| **Access Method** | Web interface only. No official API. Third-party scrapers exist (baseballr NCAA functions, collegebaseball package, ncaa-api GitHub project). |
| **Pricing** | Free to view on web |
| **Data Quality** | Official but basic. Inconsistent formatting. Data entry quality varies by institution. |
| **ToS/Commercial Use** | No explicit API terms. Scraping is technically possible but not officially sanctioned. Third-party packages (baseballr, etc.) scrape this site. |

**Strengths:** Official source. Covers all divisions and sports. Free.  
**Weaknesses:** No API. Poor data structure for programmatic access. Limited advanced metrics. Inconsistent data quality across institutions.

---

### 2.8 Sportradar

| Attribute | Details |
|-----------|---------|
| **Sports/Leagues** | NFL, NBA, MLB, NHL, NCAA Football, NCAA Basketball, NCAA Baseball, soccer, tennis, golf, motorsports, and 30+ other sports |
| **Data Types** | Real-time scores, play-by-play, box scores, player stats, team stats, standings, schedules, injuries, depth charts, odds |
| **Advanced Metrics** | Varies by sport. NFL includes Next Gen Stats (official partner). NBA includes tracking data. |
| **Historical Depth** | Varies by sport and tier. NFL: multiple decades. NBA: back to 2012-13 for detailed data [unverified]. MLB: extensive. College: more limited. |
| **Update Frequency** | Real-time during games (sub-second for some feeds) |
| **Access Method** | RESTful API. JSON/XML responses. Official SDKs available. NCAAMB API currently at v8. |
| **Pricing** | **Not publicly listed.** Enterprise pricing via sales negotiation. Overage fees: $100 per 1,000 API calls beyond plan. Estimated $1,000+/month for meaningful access [unverified]. Trial accounts available with limited calls. |
| **Data Quality** | Excellent. Official data partner for NFL, NBA, MLB, NHL, and NCAA. Industry gold standard for commercial applications. |
| **ToS/Commercial Use** | B2B licensing model. Commercial use requires paid license. Data cannot be redistributed without agreement. Competitions grouped into 9 coverage tiers. |

**Strengths:** Most comprehensive single provider. Official league partnerships. Real-time data. Production-grade reliability.  
**Weaknesses:** Expensive. Opaque pricing. Overkill for historical analysis. B2B sales process required.

---

### 2.9 SportsDataIO

| Attribute | Details |
|-----------|---------|
| **Sports/Leagues** | NFL, NBA, MLB, NHL, NCAA Football, NCAA Basketball, MMA/UFC, Soccer, Golf, Tennis, Motorsports, Esports |
| **Data Types** | Scores, odds, projections, player stats, team stats, news, images, injuries, depth charts, DFS data |
| **Advanced Metrics** | Projections and fantasy-oriented metrics. Some advanced stats per sport [unverified depth]. |
| **Historical Depth** | Decades of historical data claimed. Specific depth varies by sport [unverified]. |
| **Update Frequency** | Real-time during games. Unlimited API calls on paid plans. |
| **Access Method** | REST API. JSON/XML. Free trial available (never expires, 1,000 calls/month). |
| **Pricing** | Starts at ~$500/month. Enterprise pricing scales up significantly. Free trial for development/testing. 19th year of operation in 2026. |
| **Data Quality** | Good. Enterprise-grade infrastructure. Used by major sports media and betting platforms [unverified specific clients]. |
| **ToS/Commercial Use** | Commercial use requires paid subscription. Free trial is for development/testing only. |

**Strengths:** Covers all BookieBreaker leagues including NCAA. Includes odds and projections. Free trial for development. Unlimited API calls on paid plans.  
**Weaknesses:** Expensive for a startup/research project. NCAA data depth unclear. Pricing not transparent.

---

### 2.10 SportsDataverse R Packages (cfbfastR, hoopR, baseballr)

| Attribute | Details |
|-----------|---------|
| **Sports/Leagues** | cfbfastR: NCAA Football. hoopR: NBA + NCAA Men's Basketball. baseballr: MLB + NCAA Baseball (all divisions). |
| **Data Types** | Play-by-play, box scores, player stats, team stats, schedules, rosters, game logs, lineups |
| **Advanced Metrics** | cfbfastR: EPA, WPA, expected points. hoopR: KenPom integration, shot locations. baseballr: Statcast data, FanGraphs metrics, NCAA stats. |
| **Historical Depth** | cfbfastR: mirrors CFBD + ESPN data. hoopR: NBA play-by-play from ESPN + KenPom. baseballr NCAA functions: team stats from ~2002, player stats from ~2021. |
| **Update Frequency** | Daily during seasons. Active development (hoopR updated March 2026). |
| **Access Method** | R packages. Can be called from Python via rpy2 or subprocess. sportsdataverse-py exists as a Python port (less mature). |
| **Pricing** | Free and open-source |
| **Data Quality** | Very good. Aggregates and cleans data from multiple sources. Active community maintenance. |
| **ToS/Commercial Use** | Open-source. Data sourced from ESPN, CFBD, stats.ncaa.org, KenPom, FanGraphs, Baseball Savant -- subject to those sources' individual ToS. |

**Strengths:** Unified interface across multiple sports. Good data cleaning and standardization. EPA/WPA models for college football. baseballr has unique NCAA baseball coverage.  
**Weaknesses:** R-native (friction for Python-based BookieBreaker). sportsdataverse-py is less mature. Depends on upstream data sources that may change.

---

### 2.11 CollegeBasketballData.com (CBBD) API

| Attribute | Details |
|-----------|---------|
| **Sports/Leagues** | NCAA Men's Basketball |
| **Data Types** | Games, stats, rankings, betting lines, plays, lineups, draft data. 22 API method groups. |
| **Advanced Metrics** | [unverified -- likely includes efficiency metrics and pace-adjusted stats] |
| **Historical Depth** | Games from 2003+. Betting lines from 2013+. |
| **Update Frequency** | During season, updated regularly [unverified exact cadence] |
| **Access Method** | REST API. Python client (`pip install cbbd`, v1.26.0 as of March 2026). R client available. |
| **Pricing** | Free tier available. Premium tiers also available [unverified pricing]. |
| **Data Quality** | Good. Created by the same maintainer as CFBD (Bill Radjewski) [unverified]. Newer and less battle-tested than CFBD. |
| **ToS/Commercial Use** | API key required. Free tier for development. [unverified commercial use terms] |

**Strengths:** Purpose-built for college basketball analytics. From the same ecosystem as CFBD. Python and R clients. 20+ years of game data.  
**Weaknesses:** Newer than CFBD, less community validation. NCAA basketball only.

---

### 2.12 Additional Discovered Sources

#### collegebaseball (Python package)
- **Scope:** NCAA Baseball (all divisions)
- **Data:** Stats from stats.ncaa.org, advanced metric calculations
- **Install:** `pip install git+https://github.com/nathanblumenfeld/collegebaseball`
- **Quality:** Niche but useful. Actively maintained [unverified current status].

#### ncaa_bbStats (Python package)
- **Scope:** NCAA Baseball D-I, D-II, D-III
- **Data:** Team stats (2002-2025), player stats (2021-2025), MLB draft data (1965-2025)
- **Features:** Supports live scraping and cached CSV/JSON access
- **Install:** Available on PyPI/GitHub

#### ncaa-api (GitHub - henrygd)
- **Scope:** All NCAA sports
- **Data:** Live scores, stats, standings from ncaa.com
- **Quality:** Lightweight wrapper. Good for scores and basic stats. Limited depth.

#### BallDontLie API
- **Scope:** NBA, NFL, MLB, NHL, NCAAF, NCAAB, WNBA, MMA, World Cup
- **Data:** Stats, scores, schedules
- **Quality:** [unverified depth and reliability]

---

## 3. Per-League Comparison Matrix

### Rating Scale
- **A** = Excellent: Deep historical data, play-by-play, advanced metrics, reliable, well-maintained
- **B** = Good: Solid coverage, most data types available, minor gaps
- **C** = Adequate: Basic stats available, limited advanced metrics or history
- **D** = Poor: Minimal coverage, significant gaps, unreliable
- **--** = Not covered

### NFL

| Source | Coverage | Play-by-Play | Advanced Metrics | Historical Depth | Reliability | Overall |
|--------|----------|-------------|-----------------|-----------------|-------------|---------|
| nfl_data_py | A | A (1999+) | A (EPA, WP, CP) | A (27 seasons) | A | **A** |
| ESPN Endpoints | B | -- | C (QBR only) | C | C | **C** |
| Sports Reference | A | -- | A | A (1920+) | A (but blocked) | **N/A** |
| Sportradar | A | A | A (Next Gen) | A | A | **A** |
| SportsDataIO | B | B | B | B | B | **B** |

### NBA

| Source | Coverage | Play-by-Play | Advanced Metrics | Historical Depth | Reliability | Overall |
|--------|----------|-------------|-----------------|-----------------|-------------|---------|
| nba_api | A | A | A (tracking) | A (1946+) | B (endpoints break) | **A-** |
| ESPN Endpoints | B | -- | C | C | C | **C** |
| hoopR | A | A | B | B | B | **B+** |
| Sports Reference | A | -- | A | A | A (but blocked) | **N/A** |
| Sportradar | A | A | A | A | A | **A** |
| SportsDataIO | B | B | B | B | B | **B** |

### MLB

| Source | Coverage | Play-by-Play | Advanced Metrics | Historical Depth | Reliability | Overall |
|--------|----------|-------------|-----------------|-----------------|-------------|---------|
| pybaseball | A | A (pitch-level) | A (Statcast) | A (2008+ Statcast, 1900s+ traditional) | B (scraping) | **A** |
| ESPN Endpoints | B | -- | C | C | C | **C** |
| baseballr | A | A | A | A | B | **A** |
| Sports Reference | A | -- | A | A (1876+) | A (but blocked) | **N/A** |
| Sportradar | A | A | A | A | A | **A** |
| SportsDataIO | B | B | B | B | B | **B** |

### NCAA Football

| Source | Coverage | Play-by-Play | Advanced Metrics | Historical Depth | Reliability | Overall |
|--------|----------|-------------|-----------------|-----------------|-------------|---------|
| CFBD API | A | A (2004+) | A (PPA, EPA) | B (~20 years) | A | **A** |
| cfbfastR | A | A | A | B | A | **A** |
| ESPN Endpoints | B | -- | C | C | C | **C** |
| NCAA Portal | C | -- | D | C | C | **C-** |
| Sportradar | B | B | B | B | A | **B** |
| SportsDataIO | B | B | C | C | B | **B-** |

### NCAA Basketball

| Source | Coverage | Play-by-Play | Advanced Metrics | Historical Depth | Reliability | Overall |
|--------|----------|-------------|-----------------|-----------------|-------------|---------|
| CBBD API | B | B | B [unverified] | B (2003+) | B | **B** |
| hoopR | B | B | B (KenPom) | B | B | **B** |
| ESPN Endpoints | B | -- | C | C | C | **C** |
| NCAA Portal | C | -- | D | C | C | **C-** |
| Sportradar | B | B | B | B | A | **B** |
| SportsDataIO | B | B | C | C | B | **B-** |

### NCAA Baseball

| Source | Coverage | Play-by-Play | Advanced Metrics | Historical Depth | Reliability | Overall |
|--------|----------|-------------|-----------------|-----------------|-------------|---------|
| baseballr (NCAA) | B | C | C | C (team: 2002+, player: 2021+) | B | **B-** |
| collegebaseball pkg | C | -- | C | C | C | **C** |
| ncaa_bbStats | C | -- | D | C (2002-2025) | C | **C** |
| ESPN Endpoints | C | -- | D | D | C | **D+** |
| NCAA Portal | C | -- | D | C | C | **C-** |
| 6-4-3 Charts | B | B | C | C [unverified] | C [unverified] | **C+** |
| Sportradar | B | B | C | B | A | **B** |

---

## 4. Pro vs College Data Gap Analysis

### Overview

The data quality gap between professional and college sports is substantial and represents one of BookieBreaker's biggest challenges for NCAA predictions.

### Dimension-by-Dimension Comparison

| Dimension | Pro Leagues (NFL/NBA/MLB) | College Leagues (NCAAF/NCAAB/NCAABB) |
|-----------|--------------------------|--------------------------------------|
| **Play-by-Play Availability** | Complete, standardized, pitch/play-level. Available for 20+ years. | NCAAF: Good (CFBD, 2004+). NCAAB: Moderate (CBBD, 2003+). NCAA Baseball: Sparse and inconsistent. |
| **Tracking Data** | NFL: Next Gen Stats. NBA: Second Spectrum. MLB: Statcast (hawk-eye). Rich spatial/movement data. | Essentially nonexistent. No equivalent of Statcast/tracking for college sports. |
| **Advanced Metrics** | EPA, WPA, xBA, xwOBA, WAR, BPM, VORP -- mature, community-validated models. | NCAAF: PPA/EPA via CFBD. NCAAB: KenPom (paid), limited public advanced stats. NCAA Baseball: Virtually none publicly available. |
| **Historical Depth** | Decades to over a century (MLB since 1876, NFL since 1920). | Typically 5-20 years depending on sport and source. NCAA Baseball player stats only from 2021 in some sources. |
| **Data Standardization** | Highly standardized. Single authoritative source per league (e.g., Elias Sports Bureau, MLB Advanced Media). | Inconsistent. Data reported by individual institutions. Formatting varies. Errors more common. |
| **Player Tracking** | Universal player ID systems (MLB MLBAM ID, NBA person_id, GSIS ID for NFL). | Fragmented. No universal college player ID. Transfer portal complicates tracking. |
| **Roster Stability** | Stable. Players under multi-year contracts. Easy to model season-over-season. | High turnover (4-5 year eligibility, transfer portal, redshirts). Roster composition changes dramatically year to year. |
| **Sample Size** | NFL: 17 games/season per team. NBA: 82. MLB: 162. | NCAAF: 12-15 games. NCAAB: 30-35. NCAA Baseball: 56. Fewer games per team but more teams. |
| **Real-Time Data** | Widely available through commercial APIs and even free sources. | Limited. ESPN endpoints provide scores. CFBD Patreon tier offers real-time subscriptions for football. |
| **Update Cadence** | Same-day or next-day for all major providers. | Varies widely. Some sources lag by days. Data entry depends on individual schools. |

### Impact on BookieBreaker Models

1. **Monte Carlo Simulation Quality:** Pro league simulations can leverage rich distributional data (pitch-level, play-level with spatial coordinates). College simulations must rely on coarser game-level or drive-level data, reducing model precision.

2. **ML Feature Engineering:** Pro leagues offer hundreds of potential features (tracking metrics, advanced stats, situational splits). College features are largely limited to basic box score stats and team-level efficiency metrics.

3. **Transfer Portal Challenge:** College player movement makes historical player-level modeling unreliable. Team-level models may be more appropriate for college predictions.

4. **NCAA Baseball is the weakest link:** With no tracking data, limited play-by-play, sparse advanced metrics, and player stats only from 2021, NCAA baseball predictions will have the widest confidence intervals and lowest expected accuracy.

### Mitigation Strategies

- **For NCAA Football:** CFBD provides adequate play-by-play and EPA. Supplement with recruiting rankings and returning production metrics to proxy for roster quality.
- **For NCAA Basketball:** Combine CBBD game data with KenPom efficiency metrics (if licensed). hoopR provides ESPN play-by-play with shot locations.
- **For NCAA Baseball:** Use baseballr NCAA functions for available data. Consider the 6-4-3 Charts API for play-by-play where available. Accept wider uncertainty bounds in predictions.
- **Cross-sport:** Use team-level models rather than player-level models for all college sports due to roster instability.

---

## 5. Recommendations per League

### 5.1 NFL

| Role | Source | Rationale |
|------|--------|-----------|
| **Primary** | **nfl_data_py** | Free, comprehensive play-by-play back to 1999, industry-standard EPA/WP models, active community, Python-native. Perfect for Monte Carlo and ML. |
| **Fallback** | ESPN Endpoints | Free, no auth, good for schedules and scores if nflverse is down. |
| **Premium Option** | Sportradar | If budget allows and real-time/Next Gen Stats data is needed. |

### 5.2 NBA

| Role | Source | Rationale |
|------|--------|-----------|
| **Primary** | **nba_api** | Free, extensive endpoints, tracking data, deep historical coverage. Requires monitoring for endpoint deprecations. |
| **Fallback** | hoopR (via sportsdataverse-py or rpy2) | Provides ESPN play-by-play as backup if nba_api endpoints break. Also brings KenPom for college basketball crossover. |
| **Premium Option** | Sportradar or SportsDataIO | For production reliability and SLA guarantees. |

### 5.3 MLB

| Role | Source | Rationale |
|------|--------|-----------|
| **Primary** | **pybaseball** | Free, pitch-level Statcast data, combines Baseball Savant + FanGraphs. Essential for any serious baseball modeling. |
| **Fallback** | baseballr (via rpy2) | Provides similar data from same underlying sources. Useful if pybaseball has scraping issues. |
| **Premium Option** | Sportradar | Official MLB data partner. Production-grade reliability. |

### 5.4 NCAA Football

| Role | Source | Rationale |
|------|--------|-----------|
| **Primary** | **CFBD API** (collegefootballdata.com) | Best college football data source. Play-by-play, EPA, recruiting, team metrics. $10/month for 75K calls is very affordable. |
| **Fallback** | cfbfastR / sportsdataverse-py | Wraps both CFBD and ESPN. Provides additional data cleaning and EPA models. |
| **Supplement** | ESPN Endpoints | Free scores and schedules across all divisions. |

### 5.5 NCAA Basketball

| Role | Source | Rationale |
|------|--------|-----------|
| **Primary** | **CBBD API** (collegebasketballdata.com) | Purpose-built for college basketball. 20+ years of game data. Python client available. Same ecosystem as CFBD. |
| **Fallback** | hoopR / ESPN Endpoints | hoopR provides ESPN play-by-play and KenPom integration. ESPN endpoints cover scores and schedules. |
| **Supplement** | ncaa-api (GitHub) | Free, covers live scores and basic stats from ncaa.com. |

### 5.6 NCAA Baseball

| Role | Source | Rationale |
|------|--------|-----------|
| **Primary** | **baseballr NCAA functions** | Best available option. Team stats from 2002, game logs, rosters, play-by-play. Scrapes stats.ncaa.org. R package callable via rpy2 or subprocess. |
| **Fallback** | ncaa_bbStats / collegebaseball Python packages | Python-native alternatives for NCAA baseball data. Less mature but avoid R dependency. |
| **Supplement** | 6-4-3 Charts API | Play-by-play data with player tracking across transfers. Unique transfer-aware player ID system. [unverified reliability and coverage depth] |
| **Long-term** | Consider Sportradar or SportsDataIO | If NCAA baseball prediction accuracy is a priority and budget allows, commercial APIs may provide the most reliable college baseball data. |

---

## Appendix A: Implementation Priority

For BookieBreaker's initial build, implement data pipelines in this order based on data availability and model confidence:

1. **NFL** (nfl_data_py) -- Richest data, most predictable sport for modeling
2. **MLB** (pybaseball) -- Pitch-level Statcast data enables sophisticated models
3. **NBA** (nba_api) -- Deep tracking data, though high game variance
4. **NCAA Football** (CFBD) -- Best college data source, large market interest
5. **NCAA Basketball** (CBBD + hoopR) -- Good data, massive March Madness interest
6. **NCAA Baseball** (baseballr) -- Weakest data; implement last, with wider confidence intervals

## Appendix B: Legal and ToS Considerations

| Source | Can We Use It? | Notes |
|--------|---------------|-------|
| nfl_data_py | Yes | MIT license. Open source. |
| nba_api | Yes, with caution | Unofficial wrapper. Gray area for commercial use of NBA.com data. |
| pybaseball | Yes, with caution | Open source, but scrapes sites with their own ToS. |
| CFBD API | Yes | API key required. Free or $10/month. |
| CBBD API | Yes | API key required. Free tier available. |
| Sports Reference | **No** | Explicitly prohibits scraping, bots, and AI/ML training. |
| ESPN Endpoints | Use at own risk | Unofficial, undocumented, can break anytime. |
| NCAA Portal | Use with caution | No explicit API. Scraping not officially sanctioned. |
| Sportradar | Yes (if licensed) | Requires paid B2B agreement. |
| SportsDataIO | Yes (if licensed) | Requires paid subscription (~$500+/month). |
| baseballr NCAA | Yes, with caution | Scrapes stats.ncaa.org. No explicit prohibition found but not officially sanctioned. |

## Appendix C: Python Package Quick Reference

```bash
# NFL
pip install nfl-data-py
pip install nflreadpy  # newer alternative

# NBA
pip install nba_api

# MLB
pip install pybaseball

# NCAA Football
pip install cfbd

# NCAA Basketball
pip install cbbd

# NCAA Baseball
pip install git+https://github.com/nathanblumenfeld/collegebaseball
# or
pip install ncaa-bbStats  # [unverified exact PyPI name]

# Multi-sport (Python port of R sportsdataverse)
pip install sportsdataverse  # [unverified current availability]
```

---

*This document should be reviewed quarterly as data source availability, pricing, and terms of service change frequently. Last verified: March 2026.*
