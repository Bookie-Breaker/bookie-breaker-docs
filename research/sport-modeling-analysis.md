# Sport Modeling Analysis for BookieBreaker

## Purpose

This document analyzes the modeling considerations for each of the six leagues BookieBreaker will support. The system
uses a hybrid architecture: Monte Carlo simulation generates base probability distributions, then ML models adjust those
distributions for contextual factors that are difficult to encode directly in simulation logic (injuries,
matchup-specific tendencies, situational motivation, etc.).

This analysis will guide algorithm design for both the simulation engine and the ML adjustment layer.

---

## Table of Contents

1. [NFL](#1-nfl)
2. [NCAA Football (FBS)](#2-ncaa-football-fbs)
3. [NBA](#3-nba)
4. [NCAA Basketball (Division I)](#4-ncaa-basketball-division-i)
5. [MLB](#5-mlb)
6. [NCAA Baseball (Division I)](#6-ncaa-baseball-division-i)
7. [Cross-Sport Architecture Considerations](#7-cross-sport-architecture-considerations)

---

## 1. NFL

### Simulation Characteristics

**What makes simulation unique:** The NFL is a discrete-play sport with high variance per play. Each snap is an
independent event with a well-defined outcome (gain/loss of yards, turnover, penalty, score). This makes it highly
amenable to play-by-play Monte Carlo simulation. However, the strategic dimension is significant -- teams change
behavior based on score, time remaining, and down/distance, meaning the simulation must be state-aware rather than
purely stochastic.

**Fundamental unit being simulated:** The drive is the natural unit. Each drive is a sequence of plays starting from a
field position and ending in a score, punt, turnover, or end of half/game. Within a drive, individual plays are
simulated based on down, distance, field position, and game state.

**Game flow and state tracking:**

- Score (both teams)
- Quarter and time remaining
- Possession and field position
- Down and distance
- Timeouts remaining (each team)
- Drive history (for fatigue/momentum modeling if desired)

The simulation must handle clock management, including two-minute drill behavior, kneel-downs, and strategic timeout
usage. Fourth-down decision-making has become analytically complex and varies significantly by coaching staff -- this is
a key area where ML adjustment adds value over pure simulation.

**Computational cost:** NFL simulation is moderate. A single game has roughly 120-150 plays. A full Monte Carlo run of
10,000 simulations per game should complete in under 1 second on modern hardware with an optimized engine. The
bottleneck is not computation but rather the quality of the input distributions (play outcome probabilities conditioned
on game state).

### Key Statistical Features

**Statistics most predictive of game outcomes:**

- Offensive and defensive DVOA (Defense-adjusted Value Over Average) -- this is the single strongest composite metric
  for predicting NFL outcomes
- Turnover-adjusted EPA per play (offensive and defensive)
- Success rate (percentage of plays achieving positive EPA)
- Explosive play rate (plays of 20+ yards)
- Pressure rate and sack rate (both generating and allowing)
- Third-down conversion rate (contextually adjusted)
- Red zone efficiency (scoring TDs vs. FGs vs. turnovers)

**Key advanced metrics:**

- EPA (Expected Points Added) per play -- the foundation of modern NFL analytics
- CPOE (Completion Percentage Over Expected) -- quarterback quality
- Win Probability Added (WPA) -- useful for identifying clutch/collapse tendencies
- PFF grades -- proprietary but widely referenced; capture blocking and coverage quality that box scores miss
- Next Gen Stats (NGS) -- tracking data for separation, time to throw, etc.
- Adjusted Net Yards per Attempt (ANY/A) -- passing efficiency
- Defensive havoc rate (TFLs + forced fumbles + interceptions + PBUs per play)

**Critical contextual factors:**

- Home/away: Worth approximately 2.5-3 points historically, though this has been declining post-2020 and varies by venue
  (dome vs. outdoor, altitude in Denver, crowd noise in Seattle/Kansas City)
- Rest: Thursday night games for teams playing the prior Sunday show measurable performance drops. Bye-week advantages
  are real but modest (approximately 1 point). Teams coming off Monday Night Football playing on short rest Thursday are
  at significant disadvantage
- Weather: Wind speed above 15 mph materially reduces passing efficiency and kicking accuracy. Temperature below 20F has
  secondary effects. Rain reduces fumble security and passing efficiency. These effects compound
- Travel: West-to-east travel for early kickoffs shows measurable underperformance. London/international games introduce
  unique variance
- Divisional familiarity: Teams in the same division play twice per year; second meetings show reduced variance and are
  harder for models to beat

**Sport-specific phenomena to model:**

- **Garbage time:** Trailing teams in the fourth quarter shift to pass-heavy, aggressive play-calling. This inflates
  offensive stats and makes raw yardage misleading. The simulation must either model play-calling shifts or weight
  game-state-adjusted metrics
- **Scoring bursts:** NFL scoring is non-uniform -- teams often score in clusters. Modeling scoring as a Poisson process
  is inadequate; a modified process with momentum/state-switching is more appropriate
- **Injuries within games:** A starting quarterback injury mid-game can shift the spread by 7-14 points. The simulation
  should be able to handle mid-game parameter changes for live modeling
- **Coaching decisions:** Fourth-down aggression, two-point conversion tendencies, and end-of-half strategy vary
  enormously by coaching staff and can shift expected point totals by 1-3 points per game

### Data Availability & Quality

**Data sources:**

- Free: nflfastR (play-by-play data back to 1999, excellent quality), Pro Football Reference, NFL Game Statistics and
  Information System (via ESPN/public APIs)
- Paid: PFF ($), Next Gen Stats (limited public, full access via NFL partnership), Sharp Football Stats, TruMedia,
  Sports Info Solutions
- APIs: ESPN API (free, rate-limited), Sportradar (paid, official NFL data partner)

**Historical depth:** Play-by-play data is available back to 1999 in nflfastR with EPA calculations. Box-score data
extends much further but is less useful for simulation calibration. The 2018+ era includes more complete tracking data
(Next Gen Stats). For model training, 5-7 years of data is the practical ceiling due to rule changes (roughing the
passer emphasis in 2018, taunting in 2021, etc.) and schematic evolution.

**Data quality:** The NFL has the best data quality of any American sports league. Play-by-play data is comprehensive,
well-structured, and extensively validated by the analytics community. The main gap is pre-snap alignment data and
detailed blocking/coverage assignments, which require PFF or similar paid services.

**Hardest data to obtain:**

- Real-time injury status (the NFL's injury report system is intentionally vague; "questionable" designations carry
  minimal information)
- Practice participation details (key for weekly modeling)
- Exact weather conditions at game time (forecasts are available but imprecise)
- Referee tendencies (penalty rates by crew are available but noisy)

**Data quirks:**

- The NFL has only 272 regular-season games per year (expanded to 285 with the 17-game schedule and new teams). This is
  a tiny sample size compared to other major sports
- Rule changes create regime shifts that can invalidate older data
- The salary cap creates structural parity, meaning team quality changes substantially year-over-year
- Preseason data is nearly useless for prediction as starters play limited snaps

### Bet Type Modelability

| Bet Type             | Rating     | Notes                                                                                                                                                                                                                                             |
| -------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Spreads              | **High**   | Best-modeled NFL bet type. Closing lines are highly efficient but early-week lines can be beaten. The discrete nature of football scoring (3s and 7s) creates key numbers (3, 7, 10, 14) where small edge translates to significant value         |
| Totals (O/U)         | **High**   | Strong modelability due to pace and efficiency metrics. Weather is the main contextual factor models can exploit. Indoor/outdoor splits are well-understood but still underpriced by some books                                                   |
| Moneylines           | **Medium** | Derived from spread modeling. Value exists mainly in underdogs where the moneyline offers better payoff structure than the spread. Large favorites rarely offer value                                                                             |
| Player Props         | **Medium** | Modelable for skill positions (QB passing yards, RB rushing yards, WR receiving yards) using target share, snap count, and matchup data. Significant variance per game due to game script dependence. Books have improved considerably since 2022 |
| Team Props           | **Medium** | Team totals, first-half lines, and similar derivative markets. Modelable but require accurate game-flow simulation, not just final-score prediction                                                                                               |
| Game Props           | **Low**    | First score method, exact margin of victory, etc. Exotic enough that simulation can generate probabilities but market inefficiencies are hard to identify with limited validation data                                                            |
| Futures              | **Low**    | Super Bowl, division, conference, MVP futures. Require full-season simulation including injury risk, schedule strength, and playoff probability. Model uncertainty compounds over long horizons                                                   |
| Live/In-Game         | **Medium** | Requires real-time simulation with sub-second latency. The discrete nature of football actually helps here -- between plays, the state is well-defined. But books have invested heavily in live models                                            |
| Parlays (correlated) | **Medium** | Same-game parlays with correlated legs (e.g., high total + player over on passing yards) are modelable via joint simulation. The key advantage is that books typically price correlated parlays assuming independence, creating structural edge   |

**Why certain types are harder:** The NFL's small sample size (17 games per team per season) makes all modeling
inherently uncertain. Futures are hardest because they compound this uncertainty over many games. Game props suffer from
insufficient validation data -- how many "first score safety" outcomes can you really train on?

### Sample Size Considerations

- **Games per season:** 272 regular-season games (32 teams x 17 games / 2), plus 13 playoff games = 285 total
- **Seasons needed:** Minimum 3 seasons for base models; 5-7 for robust feature engineering. Older data must be
  carefully weighted due to rule and scheme evolution
- **Small sample risks:** With only 17 games per team, team-level statistics are inherently noisy. A team's "true"
  third-down conversion rate cannot be reliably estimated from 17 games of data. Bayesian approaches with informative
  priors (based on personnel, scheme, and coaching) are essential
- **Roster turnover:** NFL rosters turn over approximately 25-30% annually. Free agency and the draft mean that a team's
  identity changes meaningfully each offseason. This is a fundamental challenge -- by the time enough games exist to
  calibrate a model for a specific team composition, that composition is about to change. In-season model updating is
  critical

### Pro vs. College Considerations

This is addressed in the NCAA Football section below, from the college perspective.

---

## 2. NCAA Football (FBS)

### Simulation Characteristics

**What makes simulation unique:** The FBS presents the same discrete-play structure as the NFL but with far more teams
(133 in FBS as of the 2025 season), vastly unequal talent levels, and significantly less data per team. The simulation
engine's core play-by-play logic can be shared with the NFL module, but the parameterization must handle the full
spectrum from Alabama to UMass. Conference realignment adds ongoing structural complexity.

**Fundamental unit being simulated:** Drives, identical to the NFL. The play-by-play structure is the same, though the
distribution of play types differs (more RPO, more designed QB runs, wider variance in offensive tempo).

**Game flow and state tracking:** Same state variables as NFL (score, time, possession, field position, down/distance,
timeouts) with the following differences:

- Clock stops on first downs in some conferences/situations (this rule has changed recently and creates subtle modeling
  differences)
- Overtime uses a different format -- alternating possessions from the opponent's 25-yard line, with two-point
  conversion requirements after the second overtime period (rules changed in 2021)
- The play clock and game clock management differ slightly from the NFL

**Computational cost:** Per-game cost is comparable to the NFL. The challenge is scale: 133 teams means roughly 800+
games per week during the season. Full weekly simulation for all games is still computationally trivial, but model
calibration across all teams requires efficient parameterization.

### Key Statistical Features

**Statistics most predictive of outcomes:**

- SP+ (Bill Connelly's composite rating) and FPI (ESPN) -- the two dominant composite metrics for college football. SP+
  uses a drive-efficiency approach; FPI uses a game-level model with recruiting overlays
- Recruiting composite ratings (247Sports, Rivals, On3) -- in college football, talent proxied by recruiting rankings is
  a legitimate predictive feature due to the enormous talent disparity
- EPA per play (offensive and defensive) -- calculable from play-by-play data via cfbfastR
- Havoc rate (defensive disruption) and finishing drives rate
- Success rate
- Returning production percentage -- what fraction of the prior year's snaps/production returns

**Key advanced metrics:**

- EPA per play and success rate (same framework as NFL, via cfbfastR)
- IsoPPP (Isolated Points Per Play) -- explosive play measurement
- Opponent-adjusted metrics are crucial due to massive quality disparity. Raw stats against FCS opponents are nearly
  meaningless
- Transfer portal impact scores (not yet standardized; an opportunity for proprietary model development)
- PFF college grades (available but less comprehensive than NFL)

**Critical contextual factors:**

- Home-field advantage is significantly larger in college than the NFL -- historically worth 3-4 points, with specific
  venues (Death Valley at LSU, The Swamp at Florida, Autzen at Oregon) worth potentially more
- Altitude: only relevant for a handful of programs (Air Force, Colorado, BYU historically)
- Travel: more impactful than NFL because teams travel less frequently and some programs have limited travel
  infrastructure
- Noon kickoffs vs. night games: anecdotal evidence of home-field advantage being larger for night games, though this is
  confounded with opponent quality (marquee games are typically scheduled at night)
- Rivalry games: historical rivalry matchups show reduced correlation between team quality and outcome. Model should
  dampen confidence in rivalry weeks
- Motivation/look-ahead: college teams show measurable letdown in games between major opponents. This is a real
  phenomenon that ML adjustment should capture

**Sport-specific phenomena to model:**

- **Tempo extremes:** Some teams (historically Oregon, Memphis, various Air Raid teams) run 80+ plays per game while
  others run 60. This directly affects total modeling and creates correlated scoring environments
- **FCS games:** Many FBS teams schedule FCS opponents, creating 30-40 point spread games. These should be modeled
  differently and are poor calibration data
- **Garbage time:** Even more pronounced than the NFL due to larger talent gaps. Many blowouts feature extensive
  second-string play in the second half
- **Transfer portal volatility:** Since the expansion of the transfer portal (2021+), team composition can change
  dramatically between seasons and even mid-season. This is the single largest modeling challenge in college football
  today
- **Conference realignment:** The ongoing restructuring of conferences (SEC and Big Ten expansion, Pac-12 dissolution)
  changes travel patterns, rivalry dynamics, and scheduling

### Data Availability & Quality

**Data sources:**

- Free: cfbfastR (play-by-play data, good quality but less complete than NFL equivalent), Sports Reference (College
  Football Reference), 247Sports (recruiting data)
- Paid: PFF, Sportradar, ESPN analytics (some available through API), On3 (recruiting/transfer portal data)
- Community: The college football analytics community (Bill Connelly, ESPN's Bill Connelly before/after ESPN) provides
  valuable public models

**Historical depth:** Play-by-play data via cfbfastR is available from 2004 but becomes more reliable from 2014 onward.
Recruiting data from 247Sports extends to roughly 2000. Box-score data is available much further back but is less useful
for simulation parameterization.

**Pro vs. college data quality gap:** This is the most significant data challenge in college football. The gap manifests
in several ways:

- Play-by-play data has more missing values and coding errors in college
- No equivalent to NFL's Next Gen Stats tracking data for most college games
- Smaller analytics staffs mean less data validation
- Data for Group of 5 and FCS opponents is significantly sparser than for Power 4 programs
- Injury reporting is not mandatory in college (unlike the NFL), creating a major information asymmetry

**Hardest data to obtain:**

- Transfer portal movement and eligibility status -- this information is fragmented across recruiting sites and
  university announcements
- Injury information -- no mandatory reporting requirement
- Depth charts -- many programs do not publish them or publish misleading ones
- Actual weather conditions at game time for non-NFL-caliber stadiums
- Practice/scrimmage reports for team calibration during the offseason

**Data quirks:**

- FCS games create outlier data points that can distort team-level statistics if not filtered
- Conference championship games, bowl games, and now CFP games have different motivational dynamics that don't fit
  regular-season patterns
- Spring practice and early-season games occur before rosters are finalized due to fall camp attrition and late
  transfers
- The regular season is only 12 games per team, providing even less data than the NFL's 17

### Bet Type Modelability

| Bet Type             | Rating       | Notes                                                                                                                                                                                                         |
| -------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Spreads              | **High**     | The most modelable college football bet. Lines for games between teams of similar quality are beatable. Extreme spreads (30+) are also exploitable because books struggle with tail behavior                  |
| Totals (O/U)         | **Medium**   | Tempo and style mismatches create modeling opportunities. However, the quality gap between teams makes total prediction harder -- blowout dynamics are difficult to simulate accurately                       |
| Moneylines           | **Medium**   | Valuable for moderate underdogs (roughly +150 to +400 range). Heavy favorites offer negligible moneyline value. Heavy underdogs are too uncertain                                                             |
| Player Props         | **Very Low** | Offered for marquee games only. Data per player is insufficient (12-game seasons, often with garbage time). Not worth modeling systematically                                                                 |
| Team Props           | **Low**      | Limited market availability. Team totals for major games are modelable but the market is thin                                                                                                                 |
| Game Props           | **Very Low** | Minimal market availability outside major games. Insufficient data to validate models                                                                                                                         |
| Futures              | **Low**      | National championship, conference championship, and Heisman futures exist. Modelable via full-season simulation but the transfer portal and injury uncertainty make long-range prediction extremely difficult |
| Live/In-Game         | **Low**      | Limited market availability. When available, the talent gap between teams makes live modeling tricky -- a 14-point deficit means different things in Alabama vs. Vanderbilt and Vanderbilt vs. Kent State     |
| Parlays (correlated) | **Low**      | Limited same-game parlay availability. Cross-game parlays are modelable if games are correlated (e.g., weather systems affecting multiple games)                                                              |

**Why certain types are harder:** The small per-team sample size (12 games) combined with 133 teams makes individual
team modeling unreliable. Player props are nearly unmodelable because individual player performance in 12-game samples
is dominated by noise. The quality gap between teams also makes it hard to know which data to weight -- a quarterback's
stats against FCS opponents are not informative about performance against SEC defenses.

### Sample Size Considerations

- **Games per season:** Approximately 800 FBS games per regular season, plus conference championships, bowl games, and
  CFP games (roughly 870 total)
- **Per-team games:** Only 12-13 regular-season games per team
- **Seasons needed:** 3-5 seasons of data for robust modeling, but the transfer portal era (2021+) has accelerated
  roster turnover so severely that pre-portal data may need to be downweighted
- **Small sample risks:** With 12 games per team and massive opponent quality variation, team-level EPA estimates are
  unreliable. Bayesian priors based on recruiting rankings and returning production are essential. Even midway through a
  12-game season, a team's "true" strength is not well-estimated from results alone
- **Roster turnover:** College football has the highest roster turnover of any sport covered by BookieBreaker. The
  transfer portal, early NFL draft entries, recruiting classes, and graduation mean that 30-50% of meaningful
  contributors may change year to year. Some programs (USC, Colorado under Deion Sanders) have turned over 50%+ of their
  roster in a single offseason. This fundamentally limits the value of historical team-level data

### Pro vs. College Considerations

**Shared simulation logic:**

- Core play-by-play engine (down/distance state machine, drive structure)
- EPA calculation framework
- Basic score/clock/possession tracking
- Drive outcome probability modeling (touchdown, field goal, punt, turnover)

**Must differ:**

- Overtime rules (alternating possessions from the 25, not sudden death; mandatory two-point attempts after second OT)
- Clock rules (first-down clock stoppage rules differ)
- Play-calling distributions (more RPO, designed QB runs, and option concepts in college; wider tempo variance)
- Talent-gap modeling (NFL teams are far more closely matched; college simulation must handle 40-point spread games
  gracefully)
- Two-point conversion rules in overtime
- The parameterization approach itself: NFL teams can be parameterized from their own data; college teams often need
  hybrid parameterization combining team data, recruiting priors, and conference-level priors

**College-specific challenges:**

- 133 FBS teams (vs. 32 NFL teams) means the model parameter space is 4x larger with less data per entity
- Unbalanced schedules make cross-team comparison difficult without sophisticated opponent adjustment
- The transfer portal creates mid-offseason parameter shifts that no amount of historical data can predict
- Bowl/CFP games have unique motivational dynamics (opt-outs, coaching changes between regular season and bowl)
- Coaching changes can fundamentally alter a team's scheme and identity overnight

---

## 3. NBA

### Simulation Characteristics

**What makes simulation unique:** Basketball is a continuous-flow sport with discrete possessions. Unlike football,
possessions are short (typically 10-24 seconds), abundant (approximately 100 per team per game), and relatively
homogeneous in structure. This makes possession-level simulation natural, and the high number of possessions per game
means that in-game variance is lower than football. The challenge is modeling player interactions -- basketball is the
most lineup-dependent of the major sports.

**Fundamental unit being simulated:** The possession. Each possession results in one of: made field goal (2pt or 3pt),
free throws, turnover, or offensive rebound extending the possession. The simulation cycles between offensive
possessions for each team.

**Game flow and state tracking:**

- Score (both teams)
- Quarter/half and time remaining
- Possession
- Foul counts (team fouls per quarter, individual player fouls)
- Lineup on the floor (critical -- 5 players per side)
- Substitution patterns (when players enter and exit)
- Bonus/double-bonus status

The most important modeling decision is how to handle lineups. The NBA's analytical revolution has been driven by
lineup-level data, and a simulation that treats teams as monolithic units will miss crucial dynamics (e.g., bench units
getting outscored, specific matchup advantages).

**Computational cost:** NBA simulation is light per game due to the regularity of possessions. Approximately 200 total
possessions per game, with simple outcome distributions per possession. 10,000 simulations complete in well under 1
second. The computational challenge is in the lineup modeling -- tracking which lineups are on the floor and adjusting
possession parameters accordingly adds complexity but not prohibitive cost.

### Key Statistical Features

**Statistics most predictive of outcomes:**

- Net rating (offensive rating minus defensive rating) -- the single strongest predictor of NBA success
- Effective field goal percentage (eFG%) -- weights three-pointers at 1.5x
- Turnover percentage
- Offensive and defensive rebound percentage
- Free throw rate (FTA/FGA)
- The "four factors" (Dean Oliver's framework: shooting, turnovers, rebounding, free throws) remain the best compact
  summary of team strength

**Key advanced metrics:**

- Offensive and defensive rating (points per 100 possessions) -- the baseline for all NBA analytics
- Player Impact Estimate (PIE), Box Plus/Minus (BPM), and RAPTOR (FiveThirtyEight, now discontinued but methodology is
  public)
- EPM (Estimated Plus-Minus) and similar regularized adjusted plus-minus metrics -- the gold standard for individual
  player evaluation
- Player tracking data: speed, distance, touches, drives, catch-and-shoot percentage, pull-up shooting percentage,
  contested vs. open shot rates
- Lineup net rating data -- available but noisy for small-minute lineups
- Clutch stats (performance in close games in the final 5 minutes) -- commonly overfit but contain some signal

**Critical contextual factors:**

- Home/away: Worth approximately 2-3 points in the NBA, declining over time. COVID-era bubble games showed teams could
  perform without home crowds, though the effect returned post-bubble
- Rest: Back-to-back games (B2B) are the most important contextual factor in the NBA. Teams on the second night of a B2B
  show measurable declines in defensive efficiency and three-point shooting. The NBA's schedule is designed to minimize
  B2Bs but they still occur 10-15 times per team per season. Third game in four nights and four games in five nights are
  rarer but show larger effects
- Altitude: Denver (5,280 feet) has a measurable home-court advantage beyond the standard, particularly in the second
  half and in back-to-back scenarios for visitors
- Travel: Cross-country travel (east-west) has a small effect, most significant for early start times
- Schedule density: Fatigue accumulates over the season. Models should track minutes played in recent games for key
  players

**Sport-specific phenomena to model:**

- **Pace:** NBA teams play at widely varying paces (possessions per 48 minutes). Pace directly affects totals and
  scoring variance. When a fast team plays a slow team, the resulting pace is a negotiation that must be modeled
- **Three-point variance:** The modern NBA is heavily three-point-dependent. Three-point shooting has high game-to-game
  variance (a team shooting 40% from three over a season may shoot 25% or 55% in any given game). This is the primary
  driver of game-level scoring variance in the NBA
- **Foul trouble:** Key players accumulating fouls changes lineup construction and can fundamentally alter game
  dynamics. This is probabilistic and should be modeled in simulation
- **Load management/rest:** Star players are routinely rested for regular-season games, particularly in the second half
  of the season and on back-to-backs. This information is often released only hours before tip-off
- **Garbage time:** NBA games that become blowouts (20+ point leads in the fourth quarter) feature bench players and
  meaningless scoring. This must be accounted for in total modeling -- a game that is competitive should have different
  scoring dynamics than a blowout
- **Playoff adjustments:** NBA playoff series involve significant tactical adjustment. Teams that lose Game 1 adjust.
  Pace typically slows, defenses tighten, and role players' contributions become more variable. Regular-season models
  require significant recalibration for playoff prediction

### Data Availability & Quality

**Data sources:**

- Free: Basketball Reference (comprehensive box scores and advanced stats), NBA API (play-by-play, shot chart, tracking
  data -- accessible but not officially documented, subject to rate limiting and breakage), nbastatR (R package),
  Cleaning the Glass (some free content)
- Paid: Second Spectrum (the NBA's official tracking data provider), Sportradar, Synergy Sports, Cleaning the Glass
  (subscription), PBPStats
- APIs: NBA.com's stats API (undocumented but widely used), ESPN API

**Historical depth:** Box-score data extends back decades. Play-by-play data is reliable from approximately 2001 onward
via Basketball Reference. Player tracking data (SportVU, then Second Spectrum) exists from 2013-14 onward but is
proprietary. Shot chart data with coordinates is available from approximately 2001 onward.

**Data quality:** Excellent for the NBA. The league's tracking system captures player and ball movement at 25 frames per
second, generating rich spatial data. Public access to tracking data is limited, but derived metrics (speed, distance,
touches, shot quality) are available through the NBA stats API.

**Hardest data to obtain:**

- Injury status and game-time decisions -- the NBA's injury report is released daily but game-time decisions
  (particularly for stars) can be announced minutes before tip-off
- Rest decisions -- load management is not announced in advance
- Full tracking data (raw coordinates) -- Second Spectrum access requires an NBA partnership
- Referee assignments -- available but not always released far in advance; specific referees correlate with foul rates
  and over/under outcomes

**Data quirks:**

- The NBA changed the three-point line distance in 1994-95, creating a regime shift in shooting data
- The pace revolution (2015-present) makes older data less representative of modern play
- The NBA bubble (2019-20) and early COVID seasons had anomalous home/away dynamics
- All-Star break creates a natural split in the season; second-half performance differs due to fatigue and motivational
  factors

### Bet Type Modelability

| Bet Type             | Rating     | Notes                                                                                                                                                                                                                                                                                                 |
| -------------------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Spreads              | **High**   | The most liquid and efficient NBA betting market. Closing lines are very sharp, but models can find value in early lines and in games where rest/lineup information is not yet priced in                                                                                                              |
| Totals (O/U)         | **High**   | Highly modelable via pace-and-efficiency framework. Pace interaction between teams is predictable. Three-point variance is the main source of model error                                                                                                                                             |
| Moneylines           | **High**   | NBA moneylines are well-priced but the volume of games allows for systematic exploitation of small edges. Valuable for moderate underdogs                                                                                                                                                             |
| Player Props         | **High**   | The NBA is the best sport for player prop modeling. Individual players have well-established usage rates, minutes, and per-minute production. With 82-game seasons, per-player data is abundant. Matchup-level adjustments (e.g., a center's production against elite interior defense) are modelable |
| Team Props           | **Medium** | Team totals, quarter-by-quarter scoring, first-half lines. Modelable but require accurate pace and game-flow simulation                                                                                                                                                                               |
| Game Props           | **Medium** | Largest lead, race to X points, etc. More modelable than in football due to the continuous nature of basketball and the high number of scoring events                                                                                                                                                 |
| Futures              | **Medium** | Championship, MVP, and other season-long futures. The NBA has less parity than the NFL, making futures more predictable -- but also more efficiently priced by books. Season-long simulation with injury probability is valuable                                                                      |
| Live/In-Game         | **High**   | Basketball's continuous flow and frequent scoring make it ideal for live modeling. Win probability can be calculated in real-time using pace, score, and time remaining. Books have invested heavily here, but the volume of in-game state changes creates opportunities                              |
| Parlays (correlated) | **High**   | Same-game parlays are highly modelable in the NBA due to correlated outcomes (high-pace game correlates with high total and high individual scoring). This is a key market for BookieBreaker                                                                                                          |

**Why certain types are harder:** Futures require modeling over an 82-game season with injury risk as the dominant
uncertainty. Game props require fine-grained simulation of scoring patterns rather than just final outcomes. Overall,
the NBA is the most modelable sport in BookieBreaker's portfolio.

### Sample Size Considerations

- **Games per season:** 1,230 regular-season games (30 teams x 82 games / 2), plus up to 105 playoff games =
  approximately 1,335 total
- **Per-team games:** 82 regular-season games, the most of any sport here
- **Seasons needed:** 2-3 seasons for strong models. More is better but the pace/style revolution means data from before
  2015 is less representative
- **Small sample risks:** Lineup-level data is the main small-sample concern. Even over an 82-game season, most lineup
  combinations play only a few hundred minutes together, making lineup-specific metrics noisy. Two-man and three-man
  combinations are more stable than five-man lineups
- **Roster turnover:** NBA rosters are moderately stable compared to college football but less stable than MLB. Free
  agency and trades mean that approximately 30-40% of rotation players change teams annually. Mid-season trades can
  fundamentally alter team composition. The trade deadline is a critical model-update point

### Pro vs. College Considerations

Addressed in the NCAA Basketball section below.

---

## 4. NCAA Basketball (Division I)

### Simulation Characteristics

**What makes simulation unique:** NCAA basketball shares the possession-based structure of the NBA but with several
critical differences: a 30-second shot clock (vs. 24 in the NBA, changed from 35 seconds in 2015-16), two 20-minute
halves instead of four 12-minute quarters, different foul rules (team fouls per half, bonus at 7, double bonus at 10),
and an enormously larger number of teams (363 Division I teams as of 2025). The simulation must handle the full spectrum
from Gonzaga to sub-200 RPI teams.

**Fundamental unit being simulated:** Possessions, same as the NBA. However, the longer shot clock means possessions are
longer on average, resulting in fewer total possessions per game (approximately 65-70 per team per game vs.
approximately 100 in the NBA).

**Game flow and state tracking:**

- Score (both teams)
- Half and time remaining (two halves, not four quarters)
- Possession
- Team foul count (per half)
- Individual player fouls (5 fouls for disqualification vs. 6 in the NBA)
- Bonus/double-bonus status (7th/10th team foul, not 5th as in NBA)

**Computational cost:** Per-game cost is comparable to the NBA. The scale challenge is significant: 363 teams means
approximately 5,500 games per regular season. Full simulation of the NCAA tournament bracket (67 games, millions of
possible bracket outcomes) is computationally manageable but requires careful propagation of uncertainty through the
bracket.

### Key Statistical Features

**Statistics most predictive of outcomes:**

- KenPom ratings (the gold standard for college basketball analytics) -- adjusted offensive and defensive efficiency
  (points per 100 possessions, adjusted for opponent quality and game location)
- Adjusted tempo (possessions per 40 minutes)
- The four factors (effective FG%, turnover rate, offensive rebound rate, FT rate) -- same as NBA
- Barttorvik ratings (similar methodology to KenPom, freely available)
- NET rankings (the NCAA's official ranking metric, using game results, opponents, and location)

**Key advanced metrics:**

- Adjusted offensive and defensive efficiency (KenPom, Barttorvik, Haslametrics)
- Effective possession ratio (accounting for turnovers and offensive rebounds)
- Shot distribution by location (at-rim, mid-range, three-point) -- particularly important in college where shot quality
  varies more than the NBA
- Transition efficiency
- Individual player metrics: PER, usage rate, offensive rating, BPM

**Critical contextual factors:**

- Home-court advantage: Larger than the NBA, historically worth 3-4 points. Some venues (Cameron Indoor at Duke, Phog
  Allen at Kansas, Hilton Coliseum at Iowa State) are worth more. Student sections and smaller arena sizes amplify the
  effect
- Conference vs. non-conference: Early-season non-conference games (November/December) feature uncertain team quality,
  neutral-site tournaments, and teams still developing chemistry. Conference play (January-March) is more predictable
- Elevation: Relevant for programs like Air Force, Colorado, and BYU (historically) but a minor factor
- Travel: More impactful than the NBA due to less professional travel arrangements and younger players. Mid-major teams
  traveling across the country for non-conference games show measurable road performance drops
- Tournament motivation: Conference tournament "must-win" games for bubble teams have different dynamics than
  regular-season games

**Sport-specific phenomena to model:**

- **Pace interaction:** Even more important than the NBA because the pace variance is larger. Some teams play at 60
  possessions per game while others play at 75. When a fast team meets a slow team, the resulting pace is difficult to
  predict but critical for totals
- **Shot clock influence:** The 30-second shot clock allows for more deliberate offense and means that late-game
  possessions are consumed more slowly, affecting comeback probability
- **Foul dynamics:** Five fouls for disqualification (vs. six in the NBA) means foul trouble is a more significant game
  factor. A star player with three fouls in the first half is a meaningful event
- **Conference play vs. non-conference:** Teams' true quality is difficult to assess in November from 3-4 games against
  often-curated opponents. Models should heavily weight prior information (returning production, recruiting) early in
  the season
- **March Madness dynamics:** The NCAA tournament is single-elimination, which means variance is maximized. Simulation
  must propagate probability distributions through brackets accurately. "Cinderella" runs are partially explained by
  matchup dynamics and partly by irreducible variance
- **One-and-done / reclassification:** Top programs lose their best players after one year to the NBA draft. Preseason
  projections must account for incoming freshmen who have no college data

### Data Availability & Quality

**Data sources:**

- Free: KenPom (limited free, subscription for full data), Barttorvik (free and excellent), Sports Reference (College
  Basketball Reference), Haslametrics
- Paid: KenPom (subscription), Synergy Sports, PFF (limited college basketball coverage), Sportradar
- Community: Significant public analytics community around March Madness in particular (Kaggle competitions have
  produced valuable open-source models)

**Historical depth:** Box-score data is available back decades. Play-by-play data from Sports Reference is available
from approximately 2010 but becomes comprehensive around 2015. KenPom data extends to 2002. Shot-level data with
coordinates is limited outside of proprietary sources.

**Pro vs. college data quality gap:** Significant. The gap includes:

- No tracking data for the vast majority of college games
- Play-by-play data is less detailed and has more errors
- Shot location data is sparse for non-televised games (many mid-major games)
- Roster data is harder to maintain (transfers, walk-ons, redshirts)
- Game-level statistics are generally reliable; play-level statistics are inconsistent

**Hardest data to obtain:**

- Injury information (not required to be reported)
- Accurate rosters with eligibility status, scholarship vs. walk-on status
- Minutes projections for incoming freshmen
- Pre-season scrimmage or exhibition game data
- Transfer portal movements and eligibility timing

**Data quirks:**

- Early-season tournaments often feature neutral-site games that are labeled as home games for the "host" team in some
  data sources
- Conference tournament games have different motivational structures
- The 2019-20 season was truncated by COVID; the 2020-21 season had limited/no fans. These seasons require special
  treatment
- Some teams play in very small conferences (SWAC, MEAC, Big Sky, etc.) where opponent quality is so uniformly low that
  adjusted metrics are unreliable

### Bet Type Modelability

| Bet Type             | Rating       | Notes                                                                                                                                                                                                                |
| -------------------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Spreads              | **High**     | Large number of games and relatively inefficient lines (especially for smaller conferences) create opportunities. Models that correctly handle pace interactions and home-court advantages can find consistent value |
| Totals (O/U)         | **High**     | Pace-and-efficiency modeling is highly effective. The wider variance in pace across college teams means totals lines are harder for books to set precisely                                                           |
| Moneylines           | **Medium**   | Useful for underdogs in the 3-7 point spread range. March Madness moneylines are particularly valuable because public sentiment biases lines toward favorites                                                        |
| Player Props         | **Very Low** | Rarely offered outside marquee games. Per-player data is insufficient for reliable modeling. Not a target market                                                                                                     |
| Team Props           | **Low**      | Limited market. Team totals for major games are modelable                                                                                                                                                            |
| Game Props           | **Very Low** | Minimal market availability                                                                                                                                                                                          |
| Futures              | **Medium**   | National championship futures are modelable via full tournament simulation. The bracket structure creates interesting value opportunities because books sometimes misprice the probability of specific bracket paths |
| Live/In-Game         | **Low**      | Limited market availability for most games. Major games (top-25 matchups, tournament games) have live markets but they are moderately efficient                                                                      |
| Parlays (correlated) | **Medium**   | The large number of daily games (especially during conference tournaments and March Madness) creates opportunities for correlated cross-game parlays, such as when weather or venue conditions affect multiple games |

**Why certain types are harder:** The sheer number of teams and games works in BookieBreaker's favor for spread and
total modeling -- books cannot dedicate the same attention to a mid-major conference game as they can to a prime-time
NFL matchup. However, the lack of per-player data makes player props unviable, and the single-elimination tournament
format makes futures inherently high-variance.

### Sample Size Considerations

- **Games per season:** Approximately 5,500 Division I games per regular season, plus conference tournaments and the
  NCAA tournament (approximately 5,700 total)
- **Per-team games:** 28-33 games per team (regular season + conference tournament)
- **Seasons needed:** 3-5 seasons for robust models. The shot clock change (2015-16, from 35 to 30 seconds) created a
  meaningful regime shift, making pre-2015 data less comparable for tempo-related features
- **Small sample risks:** Per-team sample sizes are small (30 games) with unbalanced schedules and wide quality
  variation among opponents. KenPom and similar systems address this with Bayesian opponent adjustment, and
  BookieBreaker should adopt a similar approach. Mid-major teams that play fewer quality opponents are harder to
  evaluate accurately
- **Roster turnover:** High and accelerating. The transfer portal has transformed college basketball; teams can lose and
  gain multiple rotation players between seasons. Some programs (Kentucky under Calipari, historically) turned over
  their entire starting lineup annually. The portal era means even mid-season roster composition is not stable.
  Preseason projections based on returning production have become less reliable

### Pro vs. College Considerations

**Shared simulation logic:**

- Possession-by-possession simulation engine
- Four factors framework (shooting efficiency, turnovers, rebounding, free throws)
- Score/clock/foul tracking core
- Win probability calculation

**Must differ:**

- Shot clock: 30 seconds (college) vs. 24 seconds (NBA). This affects possession length distributions, pace, and
  late-game clock management
- Game structure: Two 20-minute halves vs. four 12-minute quarters. This changes foul dynamics and timeout strategy
- Foul rules: Disqualification at 5 fouls (college) vs. 6 (NBA). Bonus at 7th team foul per half (college) vs. 5th per
  quarter (NBA)
- Three-point line distance: College three-point line is closer (22'1.75") than the NBA (23'9"), affecting shooting
  percentages and shot distributions
- Overtime: 5-minute overtime in both, but the path to overtime and strategic play in regulation differ
- Talent distribution: NBA teams are far more balanced; college simulation must handle 30+ point spread games
- Zone defense prevalence: Zone defenses are more common and effective in college, affecting shooting efficiency
  distributions

**College-specific challenges:**

- 363 teams (vs. 30 NBA teams) -- massive parameter space with thin data per team
- Schedule imbalance makes cross-team comparison a core algorithmic challenge
- The transfer portal creates mid-offseason parameter disruption
- Incoming freshmen have zero college data, requiring recruiting-based priors
- Conference autobids in the NCAA tournament mean modeling must extend to teams outside the traditional power
  conferences

---

## 5. MLB

### Simulation Characteristics

**What makes simulation unique:** Baseball is the most simulatable sport in BookieBreaker's portfolio. The game is
fundamentally a sequence of discrete, independent plate appearances with well-defined outcomes. There is minimal "flow"
or momentum in the statistical sense -- each at-bat is largely independent of the previous one (with the exception of
baserunner state and lineup position). The game has been analytically studied for over a century, and the statistical
foundations are the strongest of any sport.

**Fundamental unit being simulated:** The plate appearance. Each plate appearance results in one of a finite set of
outcomes: strikeout, walk, hit-by-pitch, single, double, triple, home run, ground out, fly out, line out, error,
fielder's choice, sacrifice, etc. The probability of each outcome depends on the batter, the pitcher, and contextual
factors (ballpark, platoon splits, etc.). The Markov chain nature of baseball (state = baserunners + outs + inning)
makes simulation mathematically elegant.

**Game flow and state tracking:**

- Inning (1-9+) and half-inning (top/bottom)
- Outs (0, 1, 2)
- Baserunner state (8 possible configurations: bases empty through bases loaded)
- Score (both teams)
- Current batter in each lineup (batting order position)
- Current pitcher for each team
- Pitch count for the current pitcher
- Bullpen availability and usage
- Substitutions made (pinch hitters, defensive replacements)

The baserunner-outs state matrix (24 states: 8 baserunner configurations x 3 out states) combined with run expectancy
matrices provides a robust framework for evaluating individual plate appearance outcomes. This is one of the most
well-studied problems in sports analytics.

**Computational cost:** Baseball simulation is moderate. A typical game involves approximately 70-80 plate appearances.
However, the complexity comes from bullpen management simulation -- when to pull the starter, which reliever to use,
pinch-hitting decisions, and platoon matchups. A full Monte Carlo simulation of 10,000 games is computationally trivial
(well under 1 second) if bullpen decisions are pre-specified. Dynamic bullpen management (simulating when the manager
will pull the starter and which relievers will be used) adds meaningful complexity but is still fast.

### Key Statistical Features

**Statistics most predictive of outcomes:**

- Starting pitcher quality: The single most important variable in predicting MLB game outcomes. A team's win expectancy
  swings by 15-25% based on their starting pitcher matchup
- Team wOBA (weighted On-Base Average) and wRC+ (weighted Runs Created Plus) -- the best single-number offensive metrics
- Bullpen ERA/FIP/xFIP and recent workload
- Baserunning efficiency (BsR)
- Defensive metrics (DRS, OAA, UZR) -- important but noisy
- Run differential (more predictive than W-L record due to the large sample of plate appearances within games)

**Key advanced metrics:**

- wOBA and wRC+ (offense): batting metric that weights each outcome by its run value
- FIP, xFIP, SIERA (pitching): fielding-independent pitching metrics that strip out defense and luck on batted balls
- xERA, xBA, xSLG (Statcast): expected metrics based on exit velocity and launch angle, removing batted-ball luck
- Barrel rate, hard-hit rate, sprint speed (Statcast tracking data)
- Stuff+ and pitching mix modeling: evaluating individual pitch quality
- WAR (Wins Above Replacement): comprehensive player value metric with competing formulations (fWAR, bWAR, WARP)
- Run expectancy matrices by baserunner-out state
- Leverage Index for relief appearances
- Park-adjusted metrics for all of the above

**Critical contextual factors:**

- Home/away: Worth approximately 54% win probability for the home team (historically), driven partially by batting last
  (ability to walk off)
- Ballpark factors: This is the most important venue effect in any sport BookieBreaker covers. Coors Field (altitude)
  inflates run scoring by approximately 30-50%. Fenway Park (Green Monster), Yankee Stadium (short right field porch),
  and others have specific directional biases. Park factors must be calculated per metric (HR park factor, 2B park
  factor, etc.)
- Weather: Temperature and humidity directly affect ball flight distance. A 10-degree Fahrenheit increase is worth
  approximately 0.5-1 additional runs per game. Wind direction and speed at specific ballparks (especially Wrigley
  Field) can swing totals by 2+ runs
- Platoon splits: Left-handed batters vs. right-handed pitchers (and vice versa) show systematic performance
  differences. These must be modeled at the individual level
- Day/night: Some players show significant day/night splits, possibly related to visibility conditions
- Travel/fatigue: Less pronounced than basketball due to more rest days, but West Coast teams playing East Coast early
  starts show minor effects. Long road trips (10+ games) show cumulative fatigue
- Altitude: Coors Field is the extreme case, but any venue above 1,000 feet shows measurable ball-flight effects

**Sport-specific phenomena to model:**

- **Bullpen management:** The starting pitcher typically faces the lineup 2-3 times before being replaced. Reliever
  usage depends on score, inning, handedness matchups, and recent bullpen workload. Modern "opener" strategies and
  frequent pitching changes (limited by the three-batter minimum rule since 2020) must be modeled
- **Times through the order penalty (TTOP):** Starting pitchers become less effective each time through the batting
  order. This is a well-documented effect (approximately 0.010 wOBA increase per time through) and must be incorporated
  into plate appearance probability adjustments within the simulation
- **Lineup construction:** The batting order affects run production in deterministic ways. Optimal lineup construction
  is well-studied, and deviations from optimal lineups are modelable
- **Defensive shifts:** Shift restrictions (implemented in 2023) changed infield hit rates and ground ball outcomes.
  Pre-2023 data needs adjustment for shift-era effects
- **Pitch count effects:** Pitcher effectiveness degrades with increasing pitch count, particularly above 90-100
  pitches. This interacts with TTOP
- **Clutch hitting:** Statistically, "clutch hitting" (overperformance in high-leverage situations) is mostly noise
  rather than a repeatable skill. However, some pitchers show legitimate splits with runners in scoring position, and
  these should be modeled if sample sizes are adequate

### Data Availability & Quality

**Data sources:**

- Free: Baseball Reference (comprehensive, back to the 1870s), FanGraphs (advanced metrics, leaderboards, and splits),
  Baseball Savant (Statcast data from 2015+, including pitch-by-pitch tracking data with exit velocity, launch angle,
  spin rate, etc.), Retrosheet (play-by-play data back to 1921, extremely detailed)
- Paid: Sportradar, MLBAM (MLB Advanced Media) data through the Statcast system is actually largely free and public via
  Baseball Savant, making MLB the most data-rich sport by far
- APIs: pybaseball (Python package for scraping FanGraphs and Baseball Savant), MLB Stats API (officially supported)

**Historical depth:** Baseball has the deepest historical data of any sport. Box scores extend to the 1870s.
Play-by-play data (Retrosheet) is available from 1921 with increasing completeness. Pitch-by-pitch data with pitch
characteristics (PITCHf/x) exists from 2006. Statcast tracking data (exit velocity, launch angle, sprint speed) exists
from 2015 and is publicly available through Baseball Savant. This is an extraordinary data advantage.

**Data quality:** MLB data quality is the best of any sport, period. Statcast captures every pitch (spin, velocity,
movement, location) and every batted ball (exit velocity, launch angle, spray angle) with high precision. Play-by-play
data from Retrosheet has been meticulously curated by volunteers for decades. FanGraphs and Baseball Reference provide
rigorously calculated advanced metrics.

**Hardest data to obtain:**

- Real-time lineup announcements (typically released 2-3 hours before game time, but sometimes later)
- Pitcher usage plans (which reliever will be used when -- this is a key source of uncertainty)
- Injury severity and return timelines (the IL system provides minimum durations but actual return dates are uncertain)
- Weather conditions at game time (particularly wind direction and speed)
- Umpire assignments for plate umpire (announced 1-2 days in advance; individual umpires have measurable strike zone
  differences affecting walk and strikeout rates)

**Data quirks:**

- The designated hitter was adopted universally in 2022, creating a regime shift. Pre-2022 NL data includes pitcher
  batting, which depresses offensive stats
- Rule changes (pitch clock in 2023, shift restrictions in 2023, larger bases in 2023) have created multiple regime
  shifts in recent years
- Statcast metrics (xBA, xSLG, etc.) are calculated by MLB and may change methodology between seasons
- Relief pitcher data is inherently noisy due to small sample sizes per season (many relievers face only 150-250 batters
  per year)
- Minor league data quality is improving but remains significantly below MLB level

### Bet Type Modelability

| Bet Type             | Rating     | Notes                                                                                                                                                                                                                                                                                                                                               |
| -------------------- | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Spreads (run line)   | **High**   | The standard MLB run line is -1.5 / +1.5. This is highly modelable because the frequency of 1-run games is predictable from simulation. Alternative run lines (2.5, 3.5) are also modelable                                                                                                                                                         |
| Totals (O/U)         | **High**   | Extremely modelable. Run production is a function of well-understood variables (pitcher quality, offensive quality, ballpark, weather). The run-scoring environment is the most quantifiable in any sport                                                                                                                                           |
| Moneylines           | **High**   | MLB is primarily a moneyline sport (unlike football and basketball which are spread-centric). Starting pitcher matchup is the dominant factor. Models that correctly estimate team win probability can systematically exploit moneyline pricing, particularly for underdogs                                                                         |
| Player Props         | **High**   | Highly modelable for hitters (hits, total bases, RBIs, home runs) and pitchers (strikeouts, earned runs, innings pitched). Large samples of plate appearances and individual batter-vs-pitcher matchup data enable granular modeling. Strikeout props for starting pitchers are particularly well-suited to modeling                                |
| Team Props           | **Medium** | Team totals (runs scored), team hits, team strikeouts. Modelable via simulation but require accurate bullpen modeling                                                                                                                                                                                                                               |
| Game Props           | **Medium** | First team to score, total runs in specific innings (first 5 innings), method of first run. Baseball's structured inning system makes these more modelable than in continuous-flow sports                                                                                                                                                           |
| Futures              | **Medium** | Division, pennant, World Series, and MVP/Cy Young futures. The 162-game season reduces variance compared to shorter seasons, making futures more predictable. However, injuries (particularly to starting pitchers) introduce significant long-range uncertainty                                                                                    |
| Live/In-Game         | **High**   | Baseball's discrete at-bat structure makes it ideal for live modeling. The game state (inning, outs, baserunners, score, current batter, current pitcher) is fully defined between pitches. Win probability can be calculated precisely using run expectancy matrices                                                                               |
| Parlays (correlated) | **High**   | Same-game parlays have strong correlation modeling opportunities (e.g., a pitcher with many strikeouts is likely allowing fewer hits and runs; a game with high total runs correlates with player over on hits/RBIs). Weather-correlated cross-game parlays are also viable (multiple games at the same outdoor venue affected by wind/temperature) |

**Why certain types are harder:** Futures in baseball are challenging primarily because of injury unpredictability for
starting pitchers -- a team's Cy Young candidate tearing a UCL can swing their championship probability by 10-20%. Team
props require modeling not just total game scoring but the distribution across innings, which adds simulation
complexity.

### Sample Size Considerations

- **Games per season:** 2,430 regular-season games (30 teams x 162 games / 2), plus up to 43 postseason games (after
  Wild Card era expansion) = approximately 2,473 total
- **Per-team games:** 162 regular-season games -- the largest sample of any sport here
- **Seasons needed:** 2-3 seasons for strong models. However, recent rule changes (pitch clock, shift ban, universal DH)
  mean that pre-2023 data requires adjustment for some features. For Statcast-based modeling, data from 2015+ is usable
- **Small sample risks:** At the team level, 162 games provides robust estimates. The risks are at the individual level:
  relief pitchers, platoon-role players, and situational usage patterns can have small samples even over a full season.
  Rookie pitchers and September call-ups have minimal data. Bayesian priors using minor league performance (e.g.,
  translating Triple-A stats to MLB) can help
- **Roster turnover:** MLB rosters are more stable than football or basketball. Core position players and starting
  pitchers often stay with teams for 3-6 years under team control. However, bullpens are highly volatile (relievers
  change teams frequently, have short careers, and show significant year-to-year performance variance), and the bullpen
  is a critical game-outcome variable. Mid-season trades (especially at the trade deadline in late July) can
  significantly alter team composition, particularly in the bullpen

### Pro vs. College Considerations

Addressed in the NCAA Baseball section below.

---

## 6. NCAA Baseball (Division I)

### Simulation Characteristics

**What makes simulation unique:** College baseball shares the discrete plate-appearance structure of MLB but with
critical differences in game format, talent level, and data availability. The most significant structural difference is
the prevalence of multi-game series (typically three-game weekend conference series), doubleheaders, and the use of
aluminum bats. Aluminum bats dramatically change the offensive environment: exit velocities are higher, the margin for
error on pitches is smaller, and run-scoring rates are substantially elevated compared to wood-bat professional
baseball.

**Fundamental unit being simulated:** The plate appearance, same as MLB. The outcome distribution differs due to
aluminum bats (more hits, more extra-base hits, fewer strikeouts relative to MLB). The simulation engine can share the
same Markov chain structure (baserunner-outs state matrix) but must be parameterized with college-specific outcome
probabilities.

**Game flow and state tracking:**

- Same core state as MLB (inning, outs, baserunners, score, batting order position)
- Differences in game length: College baseball uses 9-inning games for standard play, but doubleheader games are often 7
  innings
- Mercy rule (run rule): Many conferences use a 10-run rule after 7 innings, which truncates blowout games
- Designated hitter usage varies: College baseball has used the DH universally for decades
- Pitching substitution patterns differ significantly from MLB (see below)

**Computational cost:** Per-game simulation is comparable to MLB. The scale is significant: approximately 300 Division I
teams means thousands of games per season. However, the data quality limitations (see below) mean that simulation
accuracy is fundamentally capped by input uncertainty rather than computational constraints.

### Key Statistical Features

**Statistics most predictive of outcomes:**

- Starting pitcher quality (strikeout rate, walk rate, HR rate allowed) -- same as MLB, this is the single most
  predictive variable
- Team batting average and on-base percentage (aluminum bats inflate these relative to MLB norms)
- Team ERA and WHIP (must be context-adjusted for conference quality and ballpark)
- Fielding is less reliable in college, making defensive errors a more significant source of scoring
- RPI (Ratings Percentage Index) -- the NCAA's historical selection metric, though its value for prediction is limited

**Key advanced metrics:**

- wOBA and FIP can be calculated for college baseball but require careful calibration to the aluminum-bat run
  environment
- Strikeout-to-walk ratio for pitchers is the most stable predictive metric due to smaller sample sizes making
  batted-ball metrics unreliable
- Boyd's World and similar sites calculate adjusted metrics (similar to KenPom for basketball) that account for opponent
  quality and run environment
- ERA+ and OPS+ (adjusted for conference and environment)
- Pitch-level data is generally not available, so Statcast-style metrics (exit velocity, launch angle, spin rate) do not
  exist for the vast majority of college games

**Critical contextual factors:**

- Home/away: Significant in college baseball, likely 3-5 points of run advantage. College baseball venues vary
  enormously in quality and atmosphere
- Ballpark factors: Even more extreme than MLB. Some college parks are bandboxes with 300-foot fences; others are
  larger. No standardized dimensions exist. Altitude effects apply to programs like Air Force
- Weather: Same physics as MLB (temperature affects ball flight, wind affects fly balls) but with more variation because
  many college programs play in less-controlled environments (no domes, less sophisticated weather management)
- Aluminum bat regulations: The NCAA periodically adjusts bat performance standards (BBCOR certification since 2011).
  Any regulation change creates a regime shift in offensive data
- Friday/Saturday/Sunday pitcher deployment: College baseball's three-game weekend series structure means teams deploy
  their three best starters in order. The Friday starter is almost always the ace, Saturday is the number two, and
  Sunday is the number three. This predictable rotation provides clear modeling structure
- Midweek games: Tuesday/Wednesday midweek games typically feature the 4th or 5th starter and have different quality
  dynamics than weekend series

**Sport-specific phenomena to model:**

- **Pitching depth:** College teams typically have a clear top-3 rotation for weekend series and a fourth starter for
  midweek games. The drop-off from Friday to Sunday starters is often much larger than in MLB (where the 1-5 rotation is
  more balanced). Bullpen depth is similarly concentrated -- most college teams have only 2-3 reliable relievers
- **Aluminum bat effects:** Higher batting averages (.270-.290 team averages vs. .240-.250 in MLB), more extra-base
  hits, and more runs per game (averaging 5-6 runs per team per game vs. 4-5 in MLB). The simulation must use
  college-specific run environment calibration
- **Mercy rule / run rule:** Games ending due to run rules truncate scoring distributions and create unusual data
  patterns. These must be handled appropriately in both data preprocessing and simulation
- **Weekend series dynamics:** The three-game series format creates dependencies between games (if the ace is used in
  relief on Saturday, he is not available Sunday; if the bullpen is taxed Friday, Saturday suffers)
- **Draft attrition:** Top college players are drafted and may sign, leaving mid-season (rare) or between seasons
  (common). This is less impactful than the transfer portal but still affects team quality projections
- **Eligibility and roster management:** College rosters are larger than MLB active rosters (35 players), and playing
  time distribution is less predictable

### Data Availability & Quality

**Data sources:**

- Free: NCAA statistics (stats.ncaa.org), Boyd's World (college baseball analytics), Baseball Reference (college section
  is limited), D1Baseball, individual conference websites
- Paid: Sportradar, Synergy (limited college baseball coverage), individual school analytics departments
- Community: The college baseball analytics community is small but growing. Warren Nolan (RPI calculations), Boyd's
  World, and Baseball America provide useful data

**Historical depth:** Box-score data is available for 10-15 years with moderate completeness. Play-by-play data is
sparse and inconsistent -- some conferences and teams provide it, many do not. Pitch-level data essentially does not
exist outside of programs that run their own TrackMan/Rapsodo systems. Historical depth is the shallowest of any sport
BookieBreaker covers.

**Pro vs. college data quality gap:** The gap between MLB and college baseball data quality is the largest of any
pro-college pair in BookieBreaker's coverage. MLB has the best data in sports; college baseball has among the worst.
There is:

- No public tracking data (no Statcast equivalent)
- Inconsistent play-by-play coverage (depends on home team's stats crew)
- No standardized advanced metrics across all teams
- Limited or no pitch-by-pitch data
- Box-score data is available but often has errors or missing fields for smaller programs

**Hardest data to obtain:**

- Play-by-play data for non-power conference games
- Pitching matchup information (which starter is going for each team on a given day)
- Injury information (even less transparent than other college sports)
- Reliable ballpark dimensions and park factors (many college fields are not well-documented)
- Weather conditions at game time

**Data quirks:**

- Run-rule games create truncated box scores that must be adjusted (a team that would have scored 15 runs in 9 innings
  instead scored 12 in 7 due to the mercy rule)
- Midweek game statistics are systematically different from weekend series statistics due to different pitching matchups
- Non-conference schedules include games against vastly inferior opponents (NAIA, Division II, etc.) that inflate
  statistics
- The College World Series (Omaha) has specific venue effects that apply only to the 8 teams that qualify
- Conference realignment affects schedule strength calculations

### Bet Type Modelability

| Bet Type             | Rating       | Notes                                                                                                                                                                                                                                                        |
| -------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Spreads (run line)   | **Low**      | Market availability is limited. When available, the higher run-scoring environment means run lines are harder to set (more variance in margin of victory)                                                                                                    |
| Totals (O/U)         | **Medium**   | The most modelable college baseball bet type. Run environment modeling (pitcher quality + aluminum bat effects + ballpark + weather) is feasible even with limited data. The higher scoring environment provides more data points per game                   |
| Moneylines           | **Medium**   | Available for most games. Starting pitcher matchup dominance makes this modelable if rotation information is available. The key challenge is the data quality gap for non-power conference teams                                                             |
| Player Props         | **Very Low** | Virtually non-existent market. Insufficient data to model even if markets existed                                                                                                                                                                            |
| Team Props           | **Very Low** | Minimal market availability                                                                                                                                                                                                                                  |
| Game Props           | **Very Low** | Minimal market availability                                                                                                                                                                                                                                  |
| Futures              | **Very Low** | College World Series futures exist but the single-elimination regional/super regional format combined with limited team-level data makes these essentially unmodelable with any meaningful edge                                                              |
| Live/In-Game         | **Very Low** | Virtually no market availability. Data latency for real-time scoring is a barrier                                                                                                                                                                            |
| Parlays (correlated) | **Low**      | Limited by overall market availability. Weekend series sweeps are a natural correlated bet (if the ace wins Friday, the team likely has a quality advantage that carries through the weekend) but this correlation is efficiently priced when books offer it |

**Why certain types are harder:** College baseball suffers from a fundamental data quality problem that caps model
accuracy. Even the most modelable bet types (moneylines, totals) are constrained by the lack of reliable advanced
metrics and play-by-play data. The market itself is thin -- fewer books offer college baseball lines, and those that do
set wider spreads to account for uncertainty.

### Sample Size Considerations

- **Games per season:** Approximately 5,000-6,000 Division I games per season (300 teams, 50-56 games per team, divided
  by 2)
- **Per-team games:** 50-56 games per team (the most per-team games of any college sport here)
- **Seasons needed:** 3-5 seasons for models, but the data quality limitations mean that additional seasons add less
  value than in other sports. The aluminum bat environment is relatively stable (unlike NFL rule changes), so older data
  is more usable -- provided the BBCOR certification standard hasn't changed
- **Small sample risks:** Individual pitcher statistics are reasonably stable over a 12-15 start season, but batter
  sample sizes (150-200 at-bats for regulars) are small enough that batting average and related metrics carry
  significant noise. Pitching strikeout and walk rates are the most stable features
- **Roster turnover:** College baseball has moderate roster turnover. Players have 4 years of eligibility (5 with a
  redshirt). The transfer portal is less impactful than in football or basketball, and the MLB draft removes top talent
  annually but in a more predictable pattern. Team cores can be more stable year-to-year than in the other college
  sports

### Pro vs. College Considerations

**Shared simulation logic:**

- Core plate appearance Markov chain engine (baserunner-outs state matrix, run expectancy framework)
- Pitcher quality modeling (strikeout rate, walk rate, HR rate)
- Lineup simulation (9-player batting order)
- Bullpen management simulation framework

**Must differ:**

- **Bat composition:** Aluminum bats fundamentally change outcome distributions. All plate appearance probability
  distributions must be separately calibrated for college. Exit velocities are higher, there are more hits, more
  extra-base hits, and more total runs
- **Run rule:** The simulation must support early termination when one team leads by 10+ runs after 7 innings. This
  affects expected run distributions and total modeling
- **Seven-inning doubleheader games:** The simulation must support variable game length
- **Pitching depth model:** College bullpens are shallower and less reliable. The simulation must handle wider variance
  in relief pitching quality
- **Weekend series dependencies:** Cross-game pitcher availability modeling is required (pitcher used in relief Saturday
  is unavailable Sunday)
- **Talent gap handling:** Like other college sports, the simulation must handle games between teams of vastly different
  quality
- **Park factors:** College parks need separate park factor calculations from MLB parks (and many college parks have
  insufficient data for reliable park factors)

**College-specific challenges:**

- 300 teams with vastly unequal data quality -- Power 4 conference teams have reasonable data; small-conference teams
  may have only box scores
- No standardized advanced metrics infrastructure
- Aluminum bat calibration is a separate modeling exercise from wood-bat MLB calibration
- Starting pitcher identification for future games is harder to determine in advance (no official probable pitcher
  listings like MLB provides)
- The market for college baseball betting is thin, limiting both the opportunity and the ability to validate models
  against efficient markets

---

## 7. Cross-Sport Architecture Considerations

### Simulation Engine Architecture

The six leagues fall into three natural simulation families:

1. **Football family (NFL + NCAA Football):** Discrete-play, drive-based simulation with down/distance state machine.
   Shared core engine with different parameterization for pro vs. college. The NFL variant handles a more constrained
   tactical space; the college variant must handle wider talent and tempo variance.

2. **Basketball family (NBA + NCAA Basketball):** Possession-based simulation with lineup tracking. Shared core engine
   with different shot clock, foul, and game structure rules. The NBA variant benefits from richer per-player data; the
   college variant requires more aggressive use of priors due to data scarcity.

3. **Baseball family (MLB + NCAA Baseball):** Plate-appearance Markov chain with baserunner-outs state tracking. Shared
   core engine with different outcome distributions (aluminum vs. wood bat). The MLB variant can leverage Statcast data
   for granular pitch/batted-ball modeling; the college variant operates primarily from box-score-level data.

### ML Adjustment Layer

The ML adjustment layer sits on top of Monte Carlo simulation results and adjusts probabilities for contextual factors.
The key design decision is what factors belong in the simulation engine vs. the ML layer.

**Factors that should live in the simulation engine:**

- Game structure and rules (clock, scoring, innings, fouls)
- Team offensive and defensive efficiency parameters
- Individual player performance distributions (where data supports it)
- Lineup and rotation effects
- Pace and tempo interactions

**Factors that should live in the ML adjustment layer:**

- Injury impact estimation (how much does losing player X shift win probability?)
- Motivational factors (rivalry games, elimination games, look-ahead spots)
- Weather effects (particularly for outdoor sports)
- Rest and travel effects
- Referee/umpire tendencies
- Market-specific adjustments (how does the public vs. sharp money split affect closing line value?)
- Coaching tendencies (fourth-down aggression, bullpen management philosophy)
- Recency weighting (how much to weight last 3 games vs. season-long data)

### Data Pipeline Priorities

Ranked by data richness and model reliability:

1. **MLB** -- Best data quality, deepest history, most modelable sport overall. Start here for model validation and
   pipeline development
2. **NBA** -- Excellent data quality, large sample size per team, highly modelable player props. Second priority
3. **NFL** -- Good data quality but small sample size. High market demand makes this a priority despite modeling
   challenges
4. **NCAA Basketball** -- Moderate data quality with strong community-built tools (KenPom, Barttorvik). Large market
   during March Madness
5. **NCAA Football** -- Good community tools but data quality is declining relative to the portal era's roster
   volatility. High market demand
6. **NCAA Baseball** -- Lowest data quality and thinnest betting markets. Lowest priority but may offer the largest
   inefficiencies due to the lack of sophisticated modeling by books

### Key Risk Factors by League

| League          | Primary Risk                      | Mitigation Strategy                                                                                |
| --------------- | --------------------------------- | -------------------------------------------------------------------------------------------------- |
| NFL             | Small sample size (17 games/team) | Heavy use of Bayesian priors; personnel-based features over team-level statistics                  |
| NCAA Football   | Transfer portal roster volatility | Recruiting/talent composite priors; rapid in-season updating; transfer impact modeling             |
| NBA             | Load management / rest decisions  | Real-time lineup scraping; injury report monitoring; rest-day prediction models                    |
| NCAA Basketball | 363-team parameter space          | Hierarchical Bayesian modeling; conference-level priors; KenPom-style opponent adjustment          |
| MLB             | Starting pitcher injury risk      | Pitch count monitoring; velocity trend tracking; IL probability models                             |
| NCAA Baseball   | Data quality floor                | Conservative modeling; focus on totals and moneylines only; power-conference-only initial coverage |

### Closing Notes

The hybrid simulation + ML architecture is well-suited to the diversity of these six leagues. The simulation engine
provides physically and statistically grounded base probabilities, while the ML layer captures the contextual
intelligence that separates good models from great ones. The three simulation families (football, basketball, baseball)
allow for code reuse while respecting the fundamental differences between sports.

The order of development should prioritize leagues where data quality and market size create the best risk-adjusted
opportunity: MLB and NBA first, NFL third (high demand despite modeling challenges), then college sports in order of
data quality and market depth.
