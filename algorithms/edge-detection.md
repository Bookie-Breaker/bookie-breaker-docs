# Edge Detection

Algorithm design for BookieBreaker's edge detection system. This document specifies how market odds are converted to implied probabilities, how expected value is calculated, how position sizing uses the Kelly criterion, and how edge quality is assessed.

---

## Table of Contents

1. [Implied Probability from Odds](#1-implied-probability-from-odds)
2. [Expected Value Calculation](#2-expected-value-calculation)
3. [Kelly Criterion for Position Sizing](#3-kelly-criterion-for-position-sizing)
4. [Edge Quality Assessment](#4-edge-quality-assessment)
5. [Parlay Correlation](#5-parlay-correlation)
6. [Edge Decay](#6-edge-decay)

---

## 1. Implied Probability from Odds

### American Odds to Raw Implied Probability

American odds encode the sportsbook's implied probability plus the vig (juice). The conversion formulas differ for favorites and underdogs.

**For favorites (negative odds, e.g., -150):**

```
implied_prob = |odds| / (|odds| + 100)
```

Example: -150 implies `150 / (150 + 100) = 150 / 250 = 0.600` (60.0%)

**For underdogs (positive odds, e.g., +150):**

```
implied_prob = 100 / (odds + 100)
```

Example: +150 implies `100 / (150 + 100) = 100 / 250 = 0.400` (40.0%)

**Decimal odds to probability (for reference):**

```
implied_prob = 1 / decimal_odds
```

**Implementation:**

```python
def american_to_implied_prob(odds: int) -> float:
    """Convert American odds to raw implied probability (includes vig)."""
    if odds < 0:
        return abs(odds) / (abs(odds) + 100)
    else:
        return 100 / (odds + 100)

def american_to_decimal(odds: int) -> float:
    """Convert American odds to decimal odds."""
    if odds < 0:
        return 1 + 100 / abs(odds)
    else:
        return 1 + odds / 100

def decimal_to_american(decimal_odds: float) -> int:
    """Convert decimal odds to American odds."""
    if decimal_odds >= 2.0:
        return round((decimal_odds - 1) * 100)
    else:
        return round(-100 / (decimal_odds - 1))
```

### Removing the Vig (Juice)

The raw implied probabilities from both sides of a market sum to more than 1.0. The excess is the vig -- the sportsbook's margin. For a standard -110/-110 market:

```
P_raw(side_A) = 110/210 = 0.5238
P_raw(side_B) = 110/210 = 0.5238
Total = 1.0476 (4.76% vig)
```

To get "true" no-vig probabilities, we must remove this excess. Three methods:

**Method 1: Multiplicative (Proportional) Removal**

Divide each probability by the total:

```
P_true(A) = P_raw(A) / (P_raw(A) + P_raw(B))
P_true(B) = P_raw(B) / (P_raw(A) + P_raw(B))
```

Example (-150/+130 market):

```
P_raw(fav) = 150/250 = 0.600
P_raw(dog) = 100/230 = 0.435
Total = 1.035 (3.5% vig)
P_true(fav) = 0.600 / 1.035 = 0.5797
P_true(dog) = 0.435 / 1.035 = 0.4203
```

Pros: simple, symmetric. Cons: assumes the vig is distributed proportionally to probability, which is not how most books set lines.

**Method 2: Additive Removal**

Subtract equal amounts from each side:

```
excess = P_raw(A) + P_raw(B) - 1.0
P_true(A) = P_raw(A) - excess / 2
P_true(B) = P_raw(B) - excess / 2
```

Pros: simple. Cons: can produce negative probabilities for extreme lines; assumes equal vig on both sides, which is unrealistic (books typically shade more toward the public side).

**Method 3: Power Method (Shin's Method)**

The theoretically most defensible approach. Assumes the sportsbook's overround is structured to protect against informed bettors. Solves for a parameter `z` (the "insider trading" fraction) such that:

```
P_true(A) = P_raw(A)^(1/z)
P_true(B) = P_raw(B)^(1/z)
where P_true(A) + P_true(B) = 1.0
```

This requires numerical solving:

```python
from scipy.optimize import brentq

def shin_devig(prob_a: float, prob_b: float) -> tuple[float, float]:
    """
    Remove vig using Shin's method (power method).
    Finds z such that prob_a^z + prob_b^z = 1.

    Args:
        prob_a: Raw implied probability of outcome A (includes vig)
        prob_b: Raw implied probability of outcome B (includes vig)

    Returns:
        (true_prob_a, true_prob_b) summing to 1.0
    """
    def equation(z):
        return prob_a ** z + prob_b ** z - 1.0

    # z is between 0 and 1; search for the root
    z = brentq(equation, 0.01, 1.0)

    true_a = prob_a ** z
    true_b = prob_b ** z

    return true_a, true_b
```

Pros: distributes vig more heavily to the favorite (consistent with how books actually set lines). Cons: slightly more complex, requires numerical solving.

**For three-way markets** (moneyline with draw possibility), extend to three outcomes:

```
P_true(A) + P_true(B) + P_true(draw) = 1.0
```

The same three methods generalize naturally.

### Recommendation

**Use the multiplicative method as the default**, with Shin's method as a configurable option.

Rationale:

- Multiplicative is the industry standard and produces results within 0.5% of Shin's method for typical vig levels (3-5%)
- Shin's method is theoretically superior but the practical difference is negligible for edge detection (our edges are typically 2-5%, far larger than the ~0.5% devig method difference)
- Multiplicative is computationally cheaper (no numerical solving)
- Use Shin's method when analyzing extreme lines (>-300/+250) where multiplicative devig error grows

---

## 2. Expected Value Calculation

### EV Formula

Expected value quantifies the average profit per unit wagered:

```
EV = (P_predicted * profit_if_win) - ((1 - P_predicted) * stake)
```

For a $100 bet at American odds:

```
If odds are positive (+150):
    profit_if_win = odds = 150
    EV = (P * 150) - ((1 - P) * 100)

If odds are negative (-150):
    profit_if_win = 100 * (100 / |odds|) = 100 * (100/150) = 66.67
    EV = (P * 66.67) - ((1 - P) * 100)
```

**General formula using decimal odds:**

```
EV = P_predicted * (decimal_odds - 1) - (1 - P_predicted)
EV = P_predicted * decimal_odds - 1
```

This gives EV as a fraction of stake. Multiply by 100 for percentage.

**Implementation:**

```python
def calculate_ev(predicted_prob: float, american_odds: int) -> float:
    """
    Calculate expected value as a percentage of stake.

    Args:
        predicted_prob: Our model's predicted probability (0-1)
        american_odds: The odds being offered

    Returns:
        EV as a fraction of stake (e.g., 0.05 = 5% EV)
    """
    decimal_odds = american_to_decimal(american_odds)
    ev = predicted_prob * decimal_odds - 1.0
    return ev


def calculate_ev_pct(predicted_prob: float, american_odds: int) -> float:
    """EV as a percentage (e.g., 5.0 for 5%)."""
    return calculate_ev(predicted_prob, american_odds) * 100
```

**Example:**

Our model predicts home team win probability = 0.58. The moneyline is -120 (implied ~54.5%).

```
decimal_odds = 1 + 100/120 = 1.833
EV = 0.58 * 1.833 - 1.0 = 1.063 - 1.0 = 0.063 (6.3% EV)
```

### Minimum EV Threshold

**Recommendation: 3% minimum EV for paper trading, with sport-specific adjustments.**

| League   | Minimum EV | Rationale                                                                     |
| -------- | ---------- | ----------------------------------------------------------------------------- |
| NFL      | 3%         | Most efficient market; require higher edge to justify                         |
| NCAA_FB  | 2%         | Less efficient lines, more opportunities at lower EV                          |
| NBA      | 3%         | Very efficient market, particularly for spreads                               |
| NCAA_BB  | 2%         | Less efficient, especially mid-major conference games                         |
| MLB      | 2.5%       | Moderate efficiency; moneyline format means EV calculation is straightforward |
| NCAA_BSB | 2%         | Least efficient market in our coverage                                        |

**Why not lower?** Below 2%, the edge is within the noise of our model's calibration error (~3% ECE target). Betting on 1% edges means we cannot distinguish genuine edges from model error. The vig also consumes ~4.5% at standard -110 pricing, so a 1% modeled edge likely has negative true EV after accounting for remaining model uncertainty.

**Why not higher?** Above 5%, opportunities become rare. In efficient markets like the NFL, a model finding consistent 5%+ edges is likely overfitting. The 2-3% range captures the sweet spot where edges are plausible and occur frequently enough for meaningful paper-trading volume.

---

## 3. Kelly Criterion for Position Sizing

### Full Kelly Formula

The Kelly criterion maximizes the long-run growth rate of a bankroll by sizing bets optimally:

```
f* = (b * p - q) / b
```

Where:

- `f*` = fraction of bankroll to wager
- `b` = net decimal odds (decimal_odds - 1). For -110: `b = 0.909`
- `p` = predicted probability of winning
- `q` = 1 - p = probability of losing

**Example:**

Predicted probability: 0.58, odds: -110 (decimal 1.909, b = 0.909)

```
f* = (0.909 * 0.58 - 0.42) / 0.909
f* = (0.527 - 0.42) / 0.909
f* = 0.107 / 0.909
f* = 0.118 (11.8% of bankroll)
```

### Why Full Kelly Is Too Aggressive

Full Kelly is mathematically optimal for maximizing log-utility growth rate, but it has severe practical problems:

1. **Variance is extreme.** Full Kelly produces a bankroll standard deviation approximately equal to the bankroll itself over moderate time horizons. A 50% drawdown is expected roughly every 20 bets even with a genuine edge.

2. **Probability estimates are imperfect.** The Kelly formula assumes perfect knowledge of `p`. Our model has calibration error of ~3%, meaning a "58% prediction" might actually be 55% or 61%. Full Kelly at a slightly overestimated probability dramatically increases ruin risk.

3. **Bankroll recovery is asymmetric.** Losing 50% of your bankroll requires a 100% gain to recover. Full Kelly creates deep drawdowns that take a long time to recover from.

**Quantifying the problem:** If our probability estimate has a standard error of 0.03 (3 percentage points), and we bet full Kelly, simulation shows:

- Probability of 50%+ drawdown within 200 bets: ~65%
- Probability of 75%+ drawdown within 200 bets: ~25%
- Expected drawdown: ~40%

### Fractional Kelly

**Recommendation: 1/4 Kelly (25% of the full Kelly fraction) for paper trading.**

```
f_actual = f* / 4
```

Using the example above: `f_actual = 0.118 / 4 = 0.0295` (2.95% of bankroll).

**Why 1/4 Kelly specifically:**

| Fraction   | Drawdown Risk (200 bets)         | Growth Rate (% of Full Kelly) | Our Choice                                |
| ---------- | -------------------------------- | ----------------------------- | ----------------------------------------- |
| Full (1/1) | ~65% chance of 50%+ drawdown     | 100%                          | Too aggressive                            |
| 1/2        | ~35% chance of 50%+ drawdown     | 75%                           | Still aggressive with imperfect estimates |
| **1/4**    | **~10% chance of 50%+ drawdown** | **~50%**                      | **Good balance**                          |
| 1/8        | ~2% chance of 50%+ drawdown      | ~25%                          | Too conservative for learning             |

At 1/4 Kelly:

- Growth rate is 50% of the theoretical maximum (acceptable for paper trading where the goal is learning, not maximizing returns)
- Drawdowns rarely exceed 30%, making it easier to evaluate model performance
- The strategy is robust to probability estimation errors of up to ~5 percentage points

**Implementation:**

```python
def kelly_fraction(
    predicted_prob: float,
    american_odds: int,
    kelly_multiplier: float = 0.25,
    max_bet_pct: float = 0.05,
) -> float:
    """
    Calculate fractional Kelly bet size.

    Args:
        predicted_prob: Model's predicted win probability
        american_odds: Odds being offered
        kelly_multiplier: Fraction of full Kelly (default 1/4)
        max_bet_pct: Maximum bet as fraction of bankroll (hard cap)

    Returns:
        Bet size as fraction of bankroll (0 if no edge)
    """
    decimal_odds = american_to_decimal(american_odds)
    b = decimal_odds - 1.0  # net odds
    p = predicted_prob
    q = 1.0 - p

    full_kelly = (b * p - q) / b

    if full_kelly <= 0:
        return 0.0  # no edge, no bet

    fractional = full_kelly * kelly_multiplier
    return min(fractional, max_bet_pct)
```

### Maximum Bet Size Cap

**Hard cap: 5% of bankroll per bet, regardless of Kelly output.**

Even with 1/4 Kelly, an unusually large perceived edge could suggest an outsized bet. The cap protects against model errors on individual games. At 5% max and typical EV, this cap rarely binds -- it would require a predicted probability > 70% at -110 odds for 1/4 Kelly to exceed 5%.

### Handling Simultaneous Bets

When multiple edges exist at the same time (e.g., 8 NBA games in an evening), the simple approach of independent Kelly sizing can lead to overexposure (betting 8 x 3% = 24% of bankroll simultaneously).

**Approach: Proportional scaling when total exposure exceeds a threshold.**

```python
def scale_simultaneous_bets(
    bets: list[dict],  # each has 'kelly_fraction' and 'game_id'
    max_total_exposure: float = 0.15,  # max 15% of bankroll at risk simultaneously
) -> list[dict]:
    """
    Scale down bet sizes proportionally when total exposure is too high.
    """
    total_exposure = sum(b['kelly_fraction'] for b in bets)

    if total_exposure <= max_total_exposure:
        return bets  # no scaling needed

    scale_factor = max_total_exposure / total_exposure

    for bet in bets:
        bet['kelly_fraction'] *= scale_factor
        bet['scaled'] = True
        bet['scale_factor'] = scale_factor

    return bets
```

**Maximum total exposure: 15% of bankroll** across all simultaneous bets. This limits the worst-case scenario (all bets lose) to a 15% drawdown in a single day.

**Correlation adjustment:** If two simultaneous bets are correlated (e.g., two games affected by the same weather system, or same-game parlay legs), reduce the combined Kelly fraction further. See Section 5 (Parlay Correlation) for correlation estimation.

---

## 4. Edge Quality Assessment

### Confidence Weighting

Not all 3% edges are equal. An edge where our model has high confidence is more valuable than one at the boundary of our calibration error.

**Edge quality score:**

```python
def edge_quality_score(
    ev_pct: float,
    prediction_confidence: float,  # width of 90% CI
    market_efficiency: float,      # 0-1 scale, higher = more efficient
    line_freshness_hours: float,
    model_calibration_error: float,
) -> float:
    """
    Composite edge quality score (0-1 scale).
    Higher = more confident in the edge.
    """
    # EV component: higher EV = higher quality (diminishing returns above 10%)
    ev_score = min(ev_pct / 10.0, 1.0)

    # Confidence component: narrower CI = higher quality
    # CI width of 0.05 (5pp) is excellent; 0.20 (20pp) is poor
    confidence_score = max(0, 1.0 - prediction_confidence / 0.20)

    # Market efficiency penalty: edges in efficient markets are more suspicious
    efficiency_penalty = 1.0 - (market_efficiency * 0.3)

    # Freshness component: stale lines are less reliable
    freshness_score = max(0, 1.0 - line_freshness_hours / 24.0)

    # Calibration component: better-calibrated model = more trustworthy edges
    calibration_score = max(0, 1.0 - model_calibration_error / 0.05)

    # Weighted combination
    quality = (
        ev_score * 0.30 +
        confidence_score * 0.25 +
        efficiency_penalty * 0.15 +
        freshness_score * 0.15 +
        calibration_score * 0.15
    )

    return round(quality, 3)
```

### Market Efficiency Rankings

Not all betting markets are equally efficient. The model should weigh edges differently based on market type:

| Market                      | Efficiency               | Notes                                                                  |
| --------------------------- | ------------------------ | ---------------------------------------------------------------------- |
| NFL spread (closing)        | Very High (0.95)         | Closing lines are extremely efficient; edges are rare and small        |
| NFL spread (opening)        | High (0.80)              | 2-3 days before game; injury and weather info not fully priced         |
| NBA spread (closing)        | Very High (0.93)         | High liquidity drives efficiency                                       |
| MLB moneyline (closing)     | High (0.88)              | Efficient but starting pitcher changes create windows                  |
| NCAA_BB spread (mid-major)  | Moderate (0.70)          | Books dedicate less attention; wider lines                             |
| NCAA_FB spread (Group of 5) | Moderate (0.65)          | Less liquid, less analyst coverage                                     |
| NCAA_BSB moneyline          | Low (0.55)               | Thin market, limited book attention                                    |
| Player props (all sports)   | Moderate (0.60-0.75)     | Newer market, less sharp action                                        |
| Same-game parlays           | Low-Moderate (0.50-0.65) | Books price assuming independence; correlation creates structural edge |

**Implication:** Finding a 3% edge in the NFL closing spread is much more noteworthy (and suspicious) than a 3% edge in a NCAA_BSB moneyline. The model should apply higher skepticism to edges in efficient markets.

### Stale Line Detection

An edge is only actionable if the line data is current. Lines can become stale due to:

- Data pipeline latency
- Sportsbook not updating
- Line data from a snapshot that is hours old

**Staleness thresholds:**

```python
def is_line_stale(
    line_timestamp: datetime,
    game_start: datetime,
    now: datetime,
) -> tuple[bool, str]:
    """
    Determine if a line is too stale to act on.

    Returns:
        (is_stale, reason)
    """
    line_age_hours = (now - line_timestamp).total_seconds() / 3600
    time_to_game_hours = (game_start - now).total_seconds() / 3600

    # Rule 1: Line older than 4 hours is always stale
    if line_age_hours > 4.0:
        return True, f"Line is {line_age_hours:.1f} hours old (max 4h)"

    # Rule 2: Within 2 hours of game time, line must be < 30 minutes old
    if time_to_game_hours < 2.0 and line_age_hours > 0.5:
        return True, f"Line is {line_age_hours*60:.0f}min old but game starts in {time_to_game_hours:.1f}h"

    # Rule 3: Within 30 minutes of game time, line must be < 5 minutes old
    if time_to_game_hours < 0.5 and line_age_hours > 0.083:
        return True, f"Line is {line_age_hours*60:.0f}min old but game starts in {time_to_game_hours*60:.0f}min"

    return False, "Line is fresh"
```

### Closing Line Value (CLV)

CLV is the single best long-term indicator of betting skill. It measures whether your bets are placed at prices better than the closing line (the final line before the game starts).

**Why CLV is the gold standard:**

The closing line is the most efficient price. It incorporates all available information from sharp bettors, public action, and book adjustments. If you consistently bet at prices better than the closing line, you are capturing genuine value -- regardless of whether individual bets win or lose. Over any reasonable sample, positive CLV implies a positive expected return.

**CLV calculation:**

```python
def calculate_clv(
    bet_odds: int,
    closing_odds: int,
) -> float:
    """
    Calculate closing line value.

    Positive CLV means you got a better price than closing.

    Returns:
        CLV as a percentage (e.g., 2.5 means 2.5% CLV)
    """
    bet_implied = american_to_implied_prob(bet_odds)
    closing_implied = american_to_implied_prob(closing_odds)

    # CLV = closing implied - bet implied
    # Positive means you bet at a lower implied prob (better price)
    clv = (closing_implied - bet_implied) * 100

    return clv
```

**Example:**

You bet home team at -105 (implied 51.2%). The line closes at -115 (implied 53.5%).

```
CLV = 53.5% - 51.2% = 2.3%
```

You got 2.3% better than the closing price -- a strong indicator of genuine edge.

**CLV benchmarks:**

| CLV (average over 100+ bets) | Assessment                                            |
| ---------------------------- | ----------------------------------------------------- |
| > 3%                         | Exceptional; likely a genuine edge                    |
| 1-3%                         | Good; consistent long-term profitability              |
| 0-1%                         | Marginal; may be profitable after vig in some markets |
| < 0%                         | No edge; model is not beating the market              |

**CLV vs. win rate:** A bettor can have a positive win rate and negative CLV (lucky) or a negative win rate and positive CLV (unlucky short-term but has a genuine edge). Over 500+ bets, CLV converges to true skill far faster than win rate.

---

## 5. Parlay Correlation

### The Independence Assumption (and Why It Is Wrong)

Standard parlay math assumes legs are independent:

```
P(parlay) = P(leg_1) * P(leg_2) * ... * P(leg_n)
```

But many parlay combinations have correlated outcomes:

- **Same-game parlays:** A team that covers the spread is more likely to have the game go over (winning teams tend to score more). A quarterback with many passing yards is correlated with the team winning.
- **Weather-affected games:** Multiple games at the same outdoor venue or in the same weather system will have correlated totals.
- **Divisional/conference games:** Games within the same division may have correlated outcomes due to shared opponent effects.
- **Back-to-back scheduling:** If Team A plays Team B on Monday and Team C on Tuesday, the Monday game outcome affects Team A's Tuesday performance.

### Correlation Coefficient Estimation

For two-leg parlays, the joint probability with correlation is:

```
P(A and B) = P(A) * P(B) + rho * sqrt(P(A) * (1-P(A)) * P(B) * (1-P(B)))
```

Where `rho` is the Pearson correlation coefficient between outcomes A and B.

**Estimating rho from historical data:**

```python
def estimate_correlation(
    outcomes_a: np.ndarray,  # binary outcomes (0/1) for leg A
    outcomes_b: np.ndarray,  # binary outcomes (0/1) for leg B
) -> float:
    """
    Estimate correlation between two binary outcome series.
    Uses phi coefficient (equivalent to Pearson for binary data).
    """
    return np.corrcoef(outcomes_a, outcomes_b)[0, 1]
```

**Common correlation estimates (from historical data analysis):**

| Parlay Type                               | Correlation (rho) | Direction                                        |
| ----------------------------------------- | ----------------- | ------------------------------------------------ |
| Same game: spread + over                  | +0.10 to +0.20    | Covering spread slightly correlated with over    |
| Same game: ML + player points over        | +0.15 to +0.25    | Team winning correlated with star scoring        |
| Same game: ML + player passing yards over | +0.20 to +0.30    | Winning team's QB likely had good passing game   |
| Cross game: same-division matchups        | +0.02 to +0.05    | Weak but nonzero                                 |
| Cross game: weather-correlated            | +0.05 to +0.15    | Wind/cold affects multiple games similarly       |
| Cross game: independent matchups          | ~0.00             | No meaningful correlation                        |
| Same game: over + first-half over         | +0.40 to +0.50    | Strong correlation (first half is part of total) |

### Adjusted Parlay EV with Correlation

**Two-leg parlay:**

```python
def correlated_parlay_ev(
    prob_a: float,         # predicted P(leg A wins)
    prob_b: float,         # predicted P(leg B wins)
    odds_a: int,           # American odds for leg A
    odds_b: int,           # American odds for leg B
    rho: float,            # estimated correlation between A and B
    parlay_odds: int | None = None,  # offered parlay odds (if SGP)
) -> dict:
    """
    Calculate EV of a correlated parlay.

    If parlay_odds is provided (SGP), compare our joint probability
    to the implied probability from the offered odds.

    If parlay_odds is None, compute the theoretical fair parlay odds
    and compare to what a book would offer assuming independence.
    """
    # True joint probability with correlation
    joint_prob = (
        prob_a * prob_b +
        rho * np.sqrt(prob_a * (1 - prob_a) * prob_b * (1 - prob_b))
    )

    # Independent joint probability (what books typically assume)
    independent_prob = prob_a * prob_b

    # If SGP odds are offered
    if parlay_odds is not None:
        implied_prob = american_to_implied_prob(parlay_odds)
        ev = joint_prob * american_to_decimal(parlay_odds) - 1.0
        edge_from_correlation = joint_prob - independent_prob

        return {
            'joint_probability': joint_prob,
            'independent_probability': independent_prob,
            'correlation_edge': edge_from_correlation,
            'implied_probability': implied_prob,
            'ev': ev,
            'ev_pct': ev * 100,
        }

    # If no SGP odds, compute theoretical parlay
    decimal_a = american_to_decimal(odds_a)
    decimal_b = american_to_decimal(odds_b)
    parlay_decimal = decimal_a * decimal_b  # standard parlay pricing
    parlay_ev = joint_prob * parlay_decimal - 1.0

    return {
        'joint_probability': joint_prob,
        'independent_probability': independent_prob,
        'correlation_edge': joint_prob - independent_prob,
        'parlay_decimal_odds': parlay_decimal,
        'ev': parlay_ev,
        'ev_pct': parlay_ev * 100,
    }
```

**Multi-leg generalization:**

For n-leg parlays with pairwise correlations, exact computation requires the multivariate normal copula or simulation. For practical purposes with small correlations:

```python
def multi_leg_parlay_prob(
    probs: list[float],
    correlations: dict[tuple[int, int], float],  # pairwise correlations
) -> float:
    """
    Approximate joint probability for a multi-leg parlay
    with pairwise correlations.

    Uses the first-order approximation:
    P(all) ≈ prod(P_i) + sum_{i<j} rho_{ij} * sqrt(P_i*(1-P_i)*P_j*(1-P_j)) * prod_{k!=i,j} P_k
    """
    n = len(probs)
    base = np.prod(probs)

    adjustment = 0.0
    for (i, j), rho in correlations.items():
        pair_adj = rho * np.sqrt(
            probs[i] * (1 - probs[i]) * probs[j] * (1 - probs[j])
        )
        # Scale by the product of all other legs
        other_prod = base / (probs[i] * probs[j]) if probs[i] * probs[j] > 0 else 0
        adjustment += pair_adj * other_prod

    return base + adjustment
```

**Important caveat:** This first-order approximation breaks down for large correlations (rho > 0.3) or many correlated legs. For same-game parlays with 3+ correlated legs, use Monte Carlo simulation to estimate the joint probability by drawing from the simulation output distributions directly.

---

## 6. Edge Decay

### The Mechanism

Betting lines move toward the true probability over time as:

1. **Sharp bettors** identify and bet on mispriced lines, causing the book to adjust
2. **New information** becomes available (injury reports, weather updates, lineup confirmations)
3. **Books copy each other**, propagating corrections across the market
4. **Public betting** sometimes moves lines away from true value (creating temporary edges) but books adjust limits to manage exposure

The result: an edge identified at market open decays toward zero as the game approaches.

### Edge Decay Model

Edge half-life varies by sport and market:

```python
def estimate_edge_remaining(
    initial_edge_pct: float,
    hours_since_detection: float,
    hours_until_game: float,
    league: str,
    market_type: str,
) -> float:
    """
    Estimate remaining edge after accounting for market correction.

    Uses exponential decay model calibrated by sport/market.
    """
    # Half-life in hours (time for edge to decay by 50%)
    half_lives = {
        # (league, market_type): half_life_hours
        ('NFL', 'SPREAD'): 24,        # NFL lines move slowly early in the week
        ('NFL', 'TOTAL'): 18,         # Totals move faster (weather sensitivity)
        ('NFL', 'MONEYLINE'): 24,
        ('NBA', 'SPREAD'): 8,         # NBA lines move fast (daily schedule)
        ('NBA', 'TOTAL'): 6,
        ('NBA', 'MONEYLINE'): 8,
        ('MLB', 'MONEYLINE'): 4,      # MLB lines move very fast (pitcher confirms)
        ('MLB', 'TOTAL'): 4,
        ('NCAA_FB', 'SPREAD'): 36,    # College lines are slower to correct
        ('NCAA_BB', 'SPREAD'): 12,
        ('NCAA_BB', 'TOTAL'): 10,
        ('NCAA_BSB', 'MONEYLINE'): 12,
        ('NCAA_BSB', 'TOTAL'): 12,
    }

    key = (league, market_type)
    half_life = half_lives.get(key, 12)  # default 12 hours

    # Exponential decay
    decay_factor = 0.5 ** (hours_since_detection / half_life)
    remaining_edge = initial_edge_pct * decay_factor

    return remaining_edge
```

### Sport-Specific Edge Decay Patterns

**NFL:**

- Lines open Sunday/Monday evening for the following week
- Monday-Wednesday: slow movement, primarily sharp action. Edges detected here have ~24-hour half-life
- Thursday-Friday: moderate movement as injury reports solidify. Half-life shortens to ~12 hours
- Saturday-Sunday: rapid movement. Within 2 hours of kickoff, edges decay with ~2-hour half-life
- **Key windows:** Sunday night (lines open) and Friday afternoon (final injury report) are the best times to identify edges

**NBA:**

- Lines open ~18-24 hours before tip-off
- Movement is fast because of daily scheduling and late-breaking rest/injury decisions
- **Key windows:** Early afternoon on game day (before injury reports) offers the best edges. Same-day lineup confirmations (~90 minutes before tip) cause rapid line movement

**MLB:**

- Lines are highly sensitive to starting pitcher confirmation (typically 3-6 hours before first pitch)
- Pre-confirmation edges have ~12-hour half-life
- Post-confirmation edges have ~4-hour half-life (market adjusts quickly to confirmed starters)
- **Key window:** Immediately after starting pitchers are confirmed but before the market fully adjusts (~30 minutes)

**NCAA (all sports):**

- Lines are generally slower to correct due to lower liquidity and less sharp action
- Mid-major and Group of 5 games may hold edges for 24-48 hours
- Power conference games behave more like their pro equivalents but still slower

### When to Bet vs. When to Re-evaluate

**Decision framework:**

```python
def should_bet_now(
    edge_pct: float,
    edge_quality: float,
    hours_until_game: float,
    league: str,
    market_type: str,
) -> str:
    """
    Recommend whether to bet immediately, wait, or pass.

    Returns: 'BET_NOW', 'WAIT', or 'PASS'
    """
    # If edge is large and quality is high, bet now
    if edge_pct >= 5.0 and edge_quality >= 0.7:
        return 'BET_NOW'

    # If game is very soon, bet now if any edge exists
    if hours_until_game < 1.0 and edge_pct >= 2.0:
        return 'BET_NOW'

    # Estimate edge remaining at game time
    remaining = estimate_edge_remaining(
        edge_pct, 0, hours_until_game, league, market_type
    )

    # If estimated remaining edge at game time is still above threshold
    if remaining >= 2.0:
        # Wait if new information is expected soon
        if _expecting_new_info(hours_until_game, league):
            return 'WAIT'
        return 'BET_NOW'

    # Edge will likely decay below threshold before game time
    if edge_pct >= 3.0:
        return 'BET_NOW'  # capture the edge while it exists

    return 'PASS'


def _expecting_new_info(hours_until_game: float, league: str) -> bool:
    """Check if new information is expected that could change the line."""
    if league == 'NFL' and hours_until_game > 48:
        return True  # injury reports still coming
    if league in ('NBA', 'NCAA_BB') and hours_until_game > 3:
        return True  # rest decisions often late
    if league in ('MLB', 'NCAA_BSB') and hours_until_game > 6:
        return True  # starting pitcher not confirmed
    return False
```

### Re-evaluation Schedule

When an edge is detected but a `WAIT` decision is made, the system should re-evaluate at these intervals:

| Time Before Game | Re-evaluation Frequency |
| ---------------- | ----------------------- |
| > 48 hours       | Every 12 hours          |
| 24-48 hours      | Every 6 hours           |
| 12-24 hours      | Every 3 hours           |
| 4-12 hours       | Every 1 hour            |
| 1-4 hours        | Every 30 minutes        |
| < 1 hour         | Every 10 minutes        |

At each re-evaluation:

1. Fetch the current line from lines-service
2. Re-run the prediction model (simulation results may be cached)
3. Recalculate EV and edge quality
4. Make a new BET_NOW/WAIT/PASS decision
