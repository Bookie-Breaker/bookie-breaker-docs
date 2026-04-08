# Betting Lines & Odds API Provider Evaluation

**Project:** BookieBreaker Sports Prediction System  
**Date:** 2026-03-30  
**Status:** Research Complete  
**Scope:** NFL, NBA, MLB, NCAA Football, NCAA Basketball, NCAA Baseball  
**Required Bet Types:** Spreads, Totals, Moneylines, Props, Futures, Live/In-Game

---

## 1. Executive Summary

### Key Findings

1. **No single provider covers all BookieBreaker requirements perfectly.** NCAA Baseball is the most significant gap — only The Odds API and OddsJam explicitly list it, and even then prop coverage is minimal.

2. **The Odds API is the best starting point** for BookieBreaker. It offers the broadest sport/league coverage including NCAA Baseball, generous free/low-cost tiers for development, and solid documentation. Its main weakness is polling-based latency (no WebSocket streaming).

3. **Pinnacle's public API was shut down in July 2025.** This is a major industry event — Pinnacle was the gold-standard sharp reference line. Providers like SharpAPI and Unabated still claim to source Pinnacle data via private arrangements, but long-term reliability is uncertain.

4. **Enterprise providers (Sportradar, TxODDS, OddsMatrix) are overkill** for BookieBreaker's prediction use case. They are designed for licensed sportsbook operators with pricing starting at $2,000+/month.

5. **For live/in-game odds**, SharpAPI (SSE streaming at sub-89ms P50) or OpticOdds (sub-second WebSocket) are the best options. The Odds API is polling-only and unsuitable for real-time in-play applications.

6. **DraftKings, FanDuel, and BetMGM do not offer public developer APIs.** All access to their odds is through third-party aggregators.

7. **Recommended strategy:** The Odds API as the primary source, SharpAPI as the live/streaming supplement, and OddsJam as the fallback for deep NCAA and prop coverage.

---

## 2. Detailed Provider Profiles

### 2.1 The Odds API (the-odds-api.com)

| Dimension           | Detail                                                                                                                     |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **Sports/Leagues**  | 70+ sports including NFL, NBA, MLB, NHL, NCAA Football, NCAA Basketball, **NCAA Baseball**, soccer, tennis, golf, politics |
| **Bet Types**       | Moneylines (h2h), spreads, totals, outrights/futures, player props, alternate spreads/totals                               |
| **Sportsbooks**     | 40+ (DraftKings, FanDuel, BetMGM, Pinnacle, Bet365, William Hill, Betfair, 1xBet, and more)                                |
| **Data Format**     | REST API (JSON). No WebSocket/streaming. Google Sheets and Excel add-ons available                                         |
| **Latency**         | Seconds (polling-based). Not suitable for sub-second live trading                                                          |
| **Pricing**         | **Free:** 500 credits/mo. **20K:** $30/mo. **100K:** $59/mo. **5M:** $119/mo. **15M:** $249/mo                             |
| **Historical Data** | Available since 2020 on paid plans. NCAA Baseball since May 2023                                                           |
| **Reliability/SLA** | No published SLA. Generally reliable based on community reports                                                            |
| **NCAA Baseball**   | **Yes** — moneylines, spreads, totals. No props. Historical from May 2023                                                  |
| **Legal/ToS**       | Aggregates publicly available odds. Standard API terms. No resale restrictions on lower tiers [unverified]                 |

**Strengths:** Broadest NCAA coverage including baseball. Most affordable entry point. Excellent documentation. Credit-based model scales well.  
**Weaknesses:** Polling only — no real-time streaming. Credit consumption can become expensive at high request volumes. Limited prop depth for college sports.

---

### 2.2 Odds-API.io

| Dimension           | Detail                                                                                                                    |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| **Sports/Leagues**  | 34 sports including NFL, NBA, NHL, MLB, soccer, tennis, esports, cricket                                                  |
| **Bet Types**       | 100+ market types including moneylines, Asian handicaps, totals, player props, correct score                              |
| **Sportsbooks**     | 265+ bookmakers worldwide (largest count among mid-tier providers)                                                        |
| **Data Format**     | REST API (all tiers). WebSocket available as premium add-on (doubles plan price)                                          |
| **Latency**         | <150ms average response time                                                                                              |
| **Pricing**         | **Free:** £0/mo (100 req/hr, 2 books). **Starter:** £99/mo. **Growth:** £179/mo. **Pro:** £229/mo. **Enterprise:** custom |
| **Reliability/SLA** | 99.9% uptime SLA                                                                                                          |
| **NCAA Coverage**   | Not explicitly listed. NFL/NBA/MLB/NHL confirmed [unverified for NCAA]                                                    |
| **Legal/ToS**       | Official Python and Node.js SDKs. MCP server for AI assistants                                                            |

**Strengths:** Highest bookmaker count (265+). Published SLA. WebSocket option. Strong SDK support.  
**Weaknesses:** NCAA coverage unclear. GBP-denominated pricing. WebSocket doubles cost. Relatively new provider compared to others.

---

### 2.3 OddsJam

| Dimension           | Detail                                                                                                           |
| ------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **Sports/Leagues**  | NFL, NBA, MLB, NHL, NCAAF, NCAAB, **NCAA Baseball**, WNBA, CFL, UFL, KBO, NPB, soccer, golf, tennis, UFC/MMA, F1 |
| **Bet Types**       | Moneylines, spreads, totals, player props, alternate markets, futures, parlays, live/in-game                     |
| **Sportsbooks**     | 150+ sportsbooks                                                                                                 |
| **Data Format**     | REST API (JSON/XML). Push stream available for real-time odds                                                    |
| **Latency**         | Sub-second. Claims "over 1 million odds per second" processing                                                   |
| **Pricing**         | Not publicly listed. Estimated $500–$1,000+/mo for API access. Consumer tools start with 7-day free trial        |
| **Reliability/SLA** | No published SLA. Established platform with large user base                                                      |
| **NCAA Baseball**   | **Yes** — listed explicitly in sports coverage. Likely limited to main markets                                   |
| **Legal/ToS**       | Originally a consumer betting tool; API is an enterprise add-on                                                  |

**Strengths:** Excellent NCAA coverage including baseball. Deep prop markets. Built-in arbitrage/+EV tools. Historical odds for backtesting. Auto-grading for bet settlement.  
**Weaknesses:** Opaque pricing. Consumer tool heritage — API may be secondary priority. Minimum cost likely $500+/mo.

---

### 2.4 Unabated

| Dimension           | Detail                                                                                                |
| ------------------- | ----------------------------------------------------------------------------------------------------- |
| **Sports/Leagues**  | NFL, NBA, MLB, NHL, WNBA, CFB, CBB, PGA, Tennis (ATP/WTA)                                             |
| **Bet Types**       | Spreads, moneylines, totals, player props, partial game odds, alternate lines, DFS pick'em lines      |
| **Sportsbooks**     | 25+ (Pinnacle, Circa, FanDuel, DraftKings, BetMGM, Bovada, regulated US + offshore)                   |
| **Data Format**     | REST API and WebSocket                                                                                |
| **Latency**         | Real-time via WebSocket. Most books update within 30 seconds. No rate limiting or throttling          |
| **Pricing**         | Starting at **$3,000/mo** for personal use. Commercial pricing higher. Bespoke integrations available |
| **Reliability/SLA** | No published SLA. 9 AM–10 PM ET support, 7 days/week                                                  |
| **NCAA Coverage**   | College Football and College Basketball confirmed. **No NCAA Baseball**                               |
| **Legal/ToS**       | No geographic restrictions. AI/bot training explicitly permitted                                      |

**Strengths:** "Unabated Line" (vig-free consensus) is a unique sharp-line product. WebSocket streaming. Includes Pinnacle data (post-shutdown, via private arrangement). No rate limits. AI training explicitly allowed.  
**Weaknesses:** Very expensive ($3,000+/mo minimum). No NCAA Baseball. Smaller sportsbook count (25+). Pinnacle access may be fragile post-shutdown.

---

### 2.5 OpticOdds

| Dimension           | Detail                                                                                                         |
| ------------------- | -------------------------------------------------------------------------------------------------------------- |
| **Sports/Leagues**  | 30+ sports. Soccer, football, basketball, tennis, cricket, esports, and more                                   |
| **Bet Types**       | Moneylines, spreads, totals, player props, alternate lines, futures, outrights, same-game parlays, custom bets |
| **Sportsbooks**     | **200+** operators (highest count among providers reviewed)                                                    |
| **Data Format**     | REST API, WebSocket push feeds, and queue-based delivery                                                       |
| **Latency**         | Sub-second (streaming feeds)                                                                                   |
| **Pricing**         | Custom enterprise quotes only. No self-serve. Expected $500–$2,000+/mo based on scope                          |
| **Reliability/SLA** | Enterprise-grade. Serves sportsbook operators directly                                                         |
| **NCAA Coverage**   | Not explicitly confirmed. Likely covers major NCAA football/basketball [unverified]                            |
| **Legal/ToS**       | Serves operators, media, DFS platforms. Requires sales engagement                                              |

**Additional Products:** Odds Screen (trading platform), Copilot (automated trading), Bet Builder (SGP engine), embeddable widgets.

**Strengths:** Highest sportsbook count (200+). Multiple delivery formats. AI consensus pricing. Sub-second latency. Comprehensive product suite.  
**Weaknesses:** No public pricing. Sales-gated access. NCAA coverage unconfirmed. Enterprise-focused — may be overbuilt for a prediction system.

---

### 2.6 SharpAPI (sharpapi.io)

| Dimension           | Detail                                                                                                           |
| ------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **Sports/Leagues**  | NFL, NBA, NHL, MLB, **NCAAB, NCAAF**, Soccer (MLS, EPL, La Liga, Serie A, Bundesliga, Ligue 1, Champions League) |
| **Bet Types**       | Moneylines, spreads, totals. +EV detection, arbitrage, middles scanning built-in                                 |
| **Sportsbooks**     | 20+ (DraftKings, FanDuel, BetMGM, Caesars, Bet365, Pinnacle). 14+ more planned                                   |
| **Data Format**     | REST API and **SSE (Server-Sent Events)** streaming. TypeScript SDK with full types                              |
| **Latency**         | P50 under 89ms for SSE streams. Pro/Sharp tiers: zero artificial delay                                           |
| **Pricing**         | **Free:** $0/mo (12 req/min, 2 books). **Hobby:** $79/mo. **Pro:** $229/mo. **Sharp:** $399/mo                   |
| **Reliability/SLA** | No published SLA. Processes 47M+ odds daily                                                                      |
| **NCAA Coverage**   | NCAAB and NCAAF confirmed. **No NCAA Baseball**                                                                  |
| **Legal/ToS**       | Standard API terms. 3-day free trial on paid tiers                                                               |

**Strengths:** Best latency at affordable price (89ms P50). Built-in +EV and arbitrage detection using Pinnacle as reference. SSE streaming without WebSocket complexity. Free tier for development. TypeScript-first SDK.  
**Weaknesses:** Smaller sportsbook count (20+). No NCAA Baseball. Relatively newer provider. Pinnacle reference line sustainability uncertain post-API-shutdown.

---

### 2.7 SportsDataIO (sportsdata.io)

| Dimension           | Detail                                                                                                                        |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **Sports/Leagues**  | NFL, NBA, MLB, NHL, College Football, College Basketball, Golf, NASCAR, Soccer, MMA, WNBA, Tennis. 100+ leagues, 1000+ events |
| **Bet Types**       | Pre-match, in-play, historical, closing lines, props, futures, SGPs, head-to-heads, exact scores                              |
| **Sportsbooks**     | DraftKings, FanDuel, BetMGM, Caesars, BetRivers, Circa, BetOnline, Fanatics, ESPN BET                                         |
| **Data Format**     | REST API (JSON). Unlimited API calls on paid plans                                                                            |
| **Latency**         | Real-time updates with line movement timestamps [specific ms not published]                                                   |
| **Pricing**         | Free trial (never expires, all endpoints). Paid plans: custom/enterprise pricing [unverified, estimated $300–$1,000+/mo]      |
| **Reliability/SLA** | Powers FanDuel, DraftKings, Fanatics, Fox Sports. Enterprise-grade                                                            |
| **NCAA Coverage**   | College Football and College Basketball confirmed. NCAA Baseball not listed [unverified]                                      |
| **Legal/ToS**       | GRid service for cross-platform ID mapping. BAKER predictive engine available                                                 |

**Additional Products:** BAKER predictive engine, Best Bets feed (true pricing, EV, ROI), settlement verification feeds, player news/injuries, Vault (historical archive), widgets.

**Strengths:** Extremely comprehensive data beyond just odds — stats, injuries, lineups, projections. Powers major operators. BAKER engine for predictions. Free trial with full access. Settlement verification.  
**Weaknesses:** Pricing not transparent. Likely expensive for full access. NCAA Baseball coverage unconfirmed. Primarily designed for operators, not hobbyist/research use.

---

### 2.8 Sportradar

| Dimension           | Detail                                                                                                          |
| ------------------- | --------------------------------------------------------------------------------------------------------------- |
| **Sports/Leagues**  | 80+ sports, 500+ leagues, 750,000+ events/year. NFL, NBA, MLB, NHL, NCAA, international                         |
| **Bet Types**       | Pre-match, live, player props, futures, probabilities                                                           |
| **Sportsbooks**     | 150+ bookmakers for odds comparison                                                                             |
| **Data Format**     | REST API (JSON). Multiple format support (decimal, American, fractional). Multilingual                          |
| **Latency**         | 15–30 seconds for odds updates [unverified — may vary by product]                                               |
| **Pricing**         | **$2,000+/mo minimum** (startup plans). Requires sales conversation and licensing agreement                     |
| **Reliability/SLA** | Enterprise SLA. Official data partner of NFL, NBA, NHL, MLB                                                     |
| **NCAA Coverage**   | Yes — extensive. Official data partnerships with college conferences [unverified on specific baseball coverage] |
| **Legal/ToS**       | Requires licensing agreement. Usage restrictions vary by contract                                               |

**Strengths:** Deepest coverage globally. Official league data partnerships. Play-by-play alongside odds. Multilingual. Industry standard for licensed operators.  
**Weaknesses:** Very expensive. Complex onboarding (sales + licensing). Latency not best-in-class. Designed for sportsbook operators, not prediction systems.

---

### 2.9 TxODDS

| Dimension           | Detail                                                                                                                 |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| **Sports/Leagues**  | 30+ sports. Soccer (350+ leagues), NBA, NFL, MLB, NCAA Football, NCAA Basketball, tennis, golf, horse/greyhound racing |
| **Bet Types**       | 100+ market types. Pre-match and live. Specific types not enumerated publicly                                          |
| **Sportsbooks**     | 250+ bookmakers                                                                                                        |
| **Data Format**     | REST API and WebSocket (JSON). Developer Hub with documentation                                                        |
| **Latency**         | **8–10ms** average (Tx FUSION ODDS — industry-leading)                                                                 |
| **Pricing**         | Custom only. Built around specific requirements. Expected enterprise-level ($2,000+/mo)                                |
| **Reliability/SLA** | 99.9% uptime on live trading feeds                                                                                     |
| **NCAA Coverage**   | NCAA Football and Basketball confirmed (Tx SCORES product — verified in-venue data). **No NCAA Baseball confirmed**    |
| **Legal/ToS**       | Enterprise clients include Bet365, William Hill, Fanatics, Flutter, Entain, Caesars                                    |

**Additional Products:** Tx FUSION ODDS (ultra-low-latency), Tx SCORES (in-venue college data), Tx LAB (5M+ historical fixtures for backtesting), Tx SOCCER ELITE, Tx LINE (blockchain-ready).

**Strengths:** Lowest latency in the industry (8–10ms). Verified in-venue college sports data. Massive historical archive. Blue-chip client list. 99.9% uptime.  
**Weaknesses:** Enterprise-only pricing. No self-serve access. NCAA Baseball unconfirmed. Designed for sportsbook operators.

---

### 2.10 OddsMatrix (EveryMatrix)

| Dimension           | Detail                                                                                                                                                                                     |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Sports/Leagues**  | 75+ sports, 20+ esports. Soccer, basketball, tennis, American football, baseball, ice hockey, cricket, golf. NFL, NBA, MLB, NHL, College Football, College Basketball listed in navigation |
| **Bet Types**       | Pre-match, in-play, outrights. Specific market types not enumerated                                                                                                                        |
| **Sportsbooks**     | 150+ bookmakers for odds sourcing                                                                                                                                                          |
| **Data Format**     | Data feeds and APIs [specific format not publicly documented]                                                                                                                              |
| **Latency**         | Claims "zero latency" for data feeds. 200,000+ live monthly events                                                                                                                         |
| **Pricing**         | Not published. One-month free trial available. Enterprise-level                                                                                                                            |
| **Reliability/SLA** | 99% automatic market settlements. 24/7 in-house trading team. Cross-checked data from multiple sources                                                                                     |
| **NCAA Coverage**   | College Football and College Basketball referenced. **No NCAA Baseball confirmed**                                                                                                         |
| **Legal/ToS**       | Part of EveryMatrix iGaming platform. Designed for B2B sportsbook operators                                                                                                                |

**Strengths:** Part of full EveryMatrix sportsbook platform. Cross-checked multi-source data. 24/7 trading team oversight. Free trial.  
**Weaknesses:** Primarily a B2B sportsbook solution, not a data API for third-party consumers. NCAA Baseball unconfirmed. Pricing opaque. Documentation limited.

---

### 2.11 DraftKings / FanDuel / BetMGM (Direct APIs)

**There are no public developer APIs from DraftKings, FanDuel, or BetMGM.**

All access to their odds data is through third-party aggregators (The Odds API, OpticOdds, SportsDataIO, OddsJam, etc.) or through unofficial scraping tools (e.g., Apify actors).

| Dimension          | Detail                                                                              |
| ------------------ | ----------------------------------------------------------------------------------- |
| **Direct API**     | **None available publicly**                                                         |
| **Access Method**  | Third-party aggregators or web scraping                                             |
| **Affiliate APIs** | Some affiliate programs offer limited odds display, but not programmatic data feeds |
| **Legal Risk**     | Scraping likely violates Terms of Service. Aggregator access is the legal path      |

**Recommendation:** Do not attempt direct scraping. Use aggregators that include these sportsbooks in their coverage (The Odds API, OpticOdds, SharpAPI, SportsDataIO all include DK/FD/BetMGM).

---

### 2.12 Pinnacle API

| Dimension          | Detail                                                                                                                     |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| **Status**         | **PUBLIC API SHUT DOWN — July 23, 2025**                                                                                   |
| **Current Access** | Bespoke arrangements only. Email api@pinnacle.com with use case description                                                |
| **Who Can Access** | Select high-value bettors, commercial partnerships, academics, pregame handicapping projects                               |
| **Impact**         | Pinnacle was the industry-standard sharp reference line. Its closure affects all +EV and arbitrage tools that relied on it |
| **Alternatives**   | Providers like SharpAPI and Unabated claim continued Pinnacle access via private deals. Sustainability uncertain           |

**Implications for BookieBreaker:** Pinnacle lines were the gold standard for true probability estimation. Their API closure means:

- Sharp line references must come through aggregators (which may lose access at any time)
- The "Unabated Line" (vig-free consensus) is an alternative fair-price benchmark
- BookieBreaker should build its own consensus model rather than depending on a single sharp book

---

## 3. Comparison Matrix

| Provider         | Price Range         | Sportsbooks | Sports | NCAA FB/BB | NCAA Baseball | Props   | Futures | Live/In-Game  | Streaming          | Latency        | Free Tier         |
| ---------------- | ------------------- | ----------- | ------ | ---------- | ------------- | ------- | ------- | ------------- | ------------------ | -------------- | ----------------- |
| **The Odds API** | $0–249/mo           | 40+         | 70+    | Yes        | **Yes**       | Yes     | Yes     | Yes (polling) | No                 | Seconds        | Yes (500 credits) |
| **Odds-API.io**  | £0–229/mo           | 265+        | 34     | Unverified | Unverified    | Yes     | Yes\*   | Yes           | WebSocket (add-on) | <150ms         | Yes (limited)     |
| **OddsJam**      | ~$500+/mo           | 150+        | 25+    | Yes        | **Yes**       | Yes     | Yes     | Yes           | Push stream        | Sub-second     | No (7-day trial)  |
| **Unabated**     | $3,000+/mo          | 25+         | 10+    | Yes        | No            | Yes     | No\*    | Yes           | WebSocket          | ~30s           | No                |
| **OpticOdds**    | Custom ($500+)      | 200+        | 30+    | Unverified | Unverified    | Yes     | Yes     | Yes           | WebSocket/Push     | Sub-second     | No                |
| **SharpAPI**     | $0–399/mo           | 20+         | 12+    | Yes        | No            | Limited | No\*    | Yes           | SSE                | <89ms P50      | Yes (2 books)     |
| **SportsDataIO** | Custom ($300+)      | 10+ named   | 15+    | Yes        | Unverified    | Yes     | Yes     | Yes           | No\*               | Real-time      | Yes (trial)       |
| **Sportradar**   | $2,000+/mo          | 150+        | 80+    | Yes        | Unverified    | Yes     | Yes     | Yes           | Yes\*              | 15–30s         | No                |
| **TxODDS**       | Custom ($2,000+)    | 250+        | 30+    | Yes        | Unverified    | Yes\*   | Yes\*   | Yes           | WebSocket          | **8–10ms**     | No                |
| **OddsMatrix**   | Custom (enterprise) | 150+        | 75+    | Yes        | Unverified    | Yes\*   | Yes\*   | Yes           | Yes\*              | "Zero latency" | 1-mo trial        |
| **Pinnacle**     | **DEPRECATED**      | 1 (self)    | N/A    | N/A        | N/A           | N/A     | N/A     | N/A           | N/A                | N/A            | N/A               |

_Asterisk indicates the feature is implied or available but not explicitly confirmed in documentation._

---

## 4. NCAA Coverage Analysis

### 4.1 NCAA Football & Basketball

Most providers cover NCAA Football (FBS) and NCAA Basketball (Division I March Madness and regular season). This is standard territory.

| Provider     | NCAAF | NCAAB | Depth (Props/Alt Lines)      |
| ------------ | ----- | ----- | ---------------------------- |
| The Odds API | Yes   | Yes   | Main markets + limited props |
| OddsJam      | Yes   | Yes   | Full props and alt markets   |
| Unabated     | Yes   | Yes   | Props, partials, alt lines   |
| SharpAPI     | Yes   | Yes   | Main markets + EV detection  |
| SportsDataIO | Yes   | Yes   | Full props and futures       |
| Sportradar   | Yes   | Yes   | Full (official data partner) |
| TxODDS       | Yes   | Yes   | In-venue verified data       |

### 4.2 NCAA Baseball — The Critical Gap

NCAA Baseball is the most underserved market across all providers evaluated. This is expected because:

- **Fewer sportsbooks offer NCAA baseball lines** compared to football/basketball
- **Lower betting volume** means less commercial incentive for data providers
- **Prop markets are essentially nonexistent** for college baseball
- **Coverage is seasonal** (February–June, CWS in June)

#### Confirmed NCAA Baseball Coverage

| Provider         | NCAA Baseball   | Markets Available                                  | Historical Data | Notes                                                 |
| ---------------- | --------------- | -------------------------------------------------- | --------------- | ----------------------------------------------------- |
| **The Odds API** | **Yes**         | Moneylines, spreads, totals                        | Since May 2023  | Best documented NCAA Baseball source                  |
| **OddsJam**      | **Yes**         | Moneylines, spreads, totals [unverified for props] | Unknown         | Listed in sports navigation. Coverage depth uncertain |
| All others       | **Unconfirmed** | —                                                  | —               | May cover it but do not explicitly list NCAA Baseball |

#### NCAA Baseball Strategy Recommendations

1. **Primary:** Use The Odds API for NCAA Baseball — it is the only provider with documented, confirmed coverage including historical data.
2. **Supplementary:** Check OddsJam API for additional sportsbook coverage of NCAA Baseball.
3. **Fallback:** Build a lightweight scraper for DraftKings/FanDuel NCAA Baseball lines (seasonal, February–June only). This carries ToS risk but is pragmatic given the data gap.
4. **Expectation management:** NCAA Baseball will have the thinnest odds coverage of any BookieBreaker market. Props and futures will likely be unavailable. Prediction models for this sport should lean more heavily on statistical data than odds-derived signals.

---

## 5. Recommendation

### 5.1 Primary Source: The Odds API

**Plan:** Start with the 100K tier ($59/mo), scale to 5M ($119/mo) as needed.

**Rationale:**

- Only provider with confirmed NCAA Baseball coverage
- Covers all six BookieBreaker target sports/leagues
- Affordable scaling from free tier through $249/mo
- Excellent documentation and community support
- Historical data since 2020 for model training and backtesting
- 40+ sportsbooks for consensus line construction

**Limitations to mitigate:**

- Polling-only (no streaming) — acceptable for pre-game prediction but not for live betting features
- Credit-based model requires careful request budgeting

### 5.2 Streaming/Live Supplement: SharpAPI

**Plan:** Hobby tier ($79/mo) for development, Pro ($229/mo) for production.

**Rationale:**

- SSE streaming at sub-89ms — best latency at this price point
- Built-in +EV and arbitrage detection saves BookieBreaker from building these
- Free tier for development and testing
- TypeScript SDK aligns well with modern stack
- NCAAF and NCAAB coverage confirmed

**Use for:** Live/in-game odds tracking, real-time line movement alerts, +EV signal generation.

### 5.3 Deep Coverage Fallback: OddsJam

**Plan:** Evaluate API during 7-day trial. Budget $500–$1,000/mo if needed.

**Rationale:**

- 150+ sportsbooks for broadest consensus
- Confirmed NCAA Baseball coverage
- Deep player prop markets
- Historical odds for backtesting
- Auto-grading for settlement verification

**Use for:** Gap-filling when The Odds API lacks specific markets, NCAA Baseball redundancy, historical odds backtesting.

### 5.4 NCAA-Specific Strategy

```
                    ┌─────────────────────────┐
                    │   BookieBreaker Odds     │
                    │     Ingestion Layer      │
                    └────────┬────────────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼──────┐ ┌────▼─────┐ ┌──────▼──────┐
     │ The Odds API  │ │ SharpAPI │ │  OddsJam    │
     │  (Primary)    │ │ (Live)   │ │ (Fallback)  │
     └───────────────┘ └──────────┘ └─────────────┘
              │              │              │
     All 6 leagues    NFL/NBA/MLB    NCAA Baseball
     Pre-game odds    NCAAF/NCAAB    Deep props
     Historical       Live streaming  Historical
     NCAA Baseball    +EV signals     Auto-grading
```

**NFL/NBA/MLB:** Fully served by The Odds API + SharpAPI. No gaps expected.

**NCAA Football/Basketball:** Well-served by all three sources. Main markets from The Odds API, live odds from SharpAPI, props from OddsJam.

**NCAA Baseball:** The Odds API as primary (confirmed coverage). OddsJam as secondary. Accept that prop and futures markets will be thin or absent. Lean on statistical models (from stats data sources) to compensate.

### 5.5 Cost Projection

| Component    | Development Phase | Production Phase           |
| ------------ | ----------------- | -------------------------- |
| The Odds API | Free–$59/mo       | $119–$249/mo               |
| SharpAPI     | Free              | $79–$229/mo                |
| OddsJam      | 7-day trial       | $500–$1,000/mo (if needed) |
| **Total**    | **$0–$59/mo**     | **$198–$1,478/mo**         |

### 5.6 Risks and Mitigations

| Risk                                          | Likelihood | Impact | Mitigation                                                                     |
| --------------------------------------------- | ---------- | ------ | ------------------------------------------------------------------------------ |
| Pinnacle data loss across all providers       | Medium     | High   | Build internal consensus model from multiple sharp books (Circa, Bet365)       |
| The Odds API credit exhaustion at scale       | Medium     | Medium | Implement aggressive caching (5-minute TTL for pre-game, 30s for live)         |
| NCAA Baseball coverage dropped                | Low        | Medium | The Odds API has offered it since 2023; low risk of removal. OddsJam as backup |
| SharpAPI is a newer provider — stability risk | Medium     | Low    | The Odds API as fallback for all SharpAPI-served markets                       |
| OddsJam pricing increases                     | Medium     | Low    | OddsJam is optional; The Odds API covers core needs                            |

### 5.7 Providers Explicitly Not Recommended

| Provider        | Reason                                                                                                                                     |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **Sportradar**  | $2,000+/mo minimum. Designed for licensed operators. Overkill for prediction system                                                        |
| **TxODDS**      | Enterprise-only. Best-in-class latency (8ms) but irrelevant for prediction use case                                                        |
| **OddsMatrix**  | B2B sportsbook platform. Not a standalone data API                                                                                         |
| **Unabated**    | $3,000+/mo minimum. Excellent product but price-prohibitive. No NCAA Baseball                                                              |
| **OpticOdds**   | Enterprise sales process. Strong product but likely $1,000+/mo and no confirmed NCAA Baseball                                              |
| **Odds-API.io** | Promising but unconfirmed NCAA coverage. GBP pricing adds friction. Consider as future alternative to The Odds API if coverage gaps emerge |

---

## Appendix A: Data Freshness Requirements by Bet Type

| Bet Type                   | Freshness Needed | Recommended Source                               |
| -------------------------- | ---------------- | ------------------------------------------------ |
| Pre-game spreads/totals/ML | 5–15 minutes     | The Odds API (polling)                           |
| Pre-game props             | 15–30 minutes    | The Odds API or OddsJam                          |
| Futures/outrights          | 1–6 hours        | The Odds API                                     |
| Live/in-game odds          | Sub-second       | SharpAPI (SSE)                                   |
| Line movement tracking     | 1–5 minutes      | The Odds API (historical) + SharpAPI (real-time) |
| Opening lines              | Daily snapshot   | The Odds API (historical)                        |

## Appendix B: Provider Contact & Documentation Links

| Provider     | Website                  | Docs                                               | Pricing Page                                 |
| ------------ | ------------------------ | -------------------------------------------------- | -------------------------------------------- |
| The Odds API | https://the-odds-api.com | https://the-odds-api.com/liveapi/guides/v4/        | https://the-odds-api.com (on homepage)       |
| Odds-API.io  | https://odds-api.io      | https://docs.odds-api.io [unverified]              | https://odds-api.io (on homepage)            |
| OddsJam      | https://oddsjam.com      | https://oddsjam.com/odds-api                       | Contact sales                                |
| Unabated     | https://unabated.com     | https://unabated.com/get-unabated-api              | Contact sales ($3,000+/mo)                   |
| OpticOdds    | https://opticodds.com    | https://developer.opticodds.com                    | https://opticodds.com/pricing (contact form) |
| SharpAPI     | https://sharpapi.io      | https://docs.sharpapi.io                           | https://sharpapi.io (on homepage)            |
| SportsDataIO | https://sportsdata.io    | https://sportsdata.io/developers/api-documentation | https://sportsdata.io (contact sales)        |
| Sportradar   | https://sportradar.com   | https://developer.sportradar.com                   | https://marketplace.sportradar.com           |
| TxODDS       | https://txodds.net       | https://txodds.net/developer-hub/                  | Contact sales                                |
| OddsMatrix   | https://oddsmatrix.com   | N/A (B2B only)                                     | Contact sales                                |

---

_This document was compiled from provider websites, public documentation, and web research as of March 2026. Pricing and coverage are subject to change. Items marked [unverified] could not be confirmed from primary sources and should be validated before procurement decisions._
