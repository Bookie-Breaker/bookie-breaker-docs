# Simulation Algorithms

Algorithm design for BookieBreaker's Monte Carlo simulation framework. This document specifies the sport-agnostic simulation runner and the per-sport simulation plugins that model game mechanics for all 6 supported leagues.

---

## Table of Contents

1. [Sport-Agnostic Framework](#1-sport-agnostic-framework)
2. [Football Plugin (NFL + NCAA_FB)](#2-football-plugin-nfl--ncaa_fb)
3. [Basketball Plugin (NBA + NCAA_BB)](#3-basketball-plugin-nba--ncaa_bb)
4. [Baseball Plugin (MLB + NCAA_BSB)](#4-baseball-plugin-mlb--ncaa_bsb)

---

## 1. Sport-Agnostic Framework

### GameSimulator Interface

Every sport plugin implements a common interface that the Monte Carlo runner consumes. This decouples the simulation orchestration from sport-specific logic.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
import numpy as np

@dataclass
class GameResult:
    """Result of a single simulated game."""
    home_score: int
    away_score: int
    # Sport-specific metadata (e.g., innings pitched, quarters)
    metadata: dict

class GameSimulator(ABC):
    """Interface that all sport-specific simulation plugins implement."""

    @abstractmethod
    def set_parameters(self, home_params: dict, away_params: dict, context: dict) -> None:
        """
        Load team/player parameters from statistics-service data.

        Args:
            home_params: Home team stats (offensive/defensive ratings, pace, etc.)
            away_params: Away team stats
            context: Venue info, dome/outdoor, surface, neutral site flag
        """
        pass

    @abstractmethod
    def simulate_game(self, rng: np.random.Generator) -> GameResult:
        """
        Simulate one complete game using the loaded parameters.
        Must be deterministic given the same rng state.

        Args:
            rng: NumPy random generator for reproducibility

        Returns:
            GameResult with final scores and metadata
        """
        pass

    @abstractmethod
    def get_sport(self) -> str:
        """Return sport identifier: 'FOOTBALL', 'BASKETBALL', or 'BASEBALL'"""
        pass

    @abstractmethod
    def get_league(self) -> str:
        """Return league identifier: 'NFL', 'NCAA_FB', 'NBA', 'NCAA_BB', 'MLB', 'NCAA_BSB'"""
        pass
```

### Monte Carlo Runner

The runner accepts any `GameSimulator`, runs N iterations, and collects outcome distributions. It is entirely sport-agnostic.

```python
@dataclass
class SimulationOutput:
    """Aggregated output from N simulation iterations."""
    iterations_run: int
    converged: bool
    convergence_iteration: int | None  # iteration at which convergence was reached

    # Score distributions (arrays of length iterations_run)
    home_scores: np.ndarray   # int array
    away_scores: np.ndarray   # int array

    # Derived distributions
    margins: np.ndarray       # home_score - away_score
    totals: np.ndarray        # home_score + away_score

    # Summary probabilities
    home_win_prob: float
    away_win_prob: float
    draw_prob: float          # regulation only (relevant for some markets)

    # Distribution statistics
    margin_mean: float
    margin_std: float
    total_mean: float
    total_std: float

    # Pre-computed spread cover probabilities at common lines
    spread_covers: dict[float, float]    # {line: P(home covers)}
    total_overs: dict[float, float]      # {line: P(over)}

    # Metadata
    parameters_hash: str
    elapsed_ms: float


def run_monte_carlo(
    simulator: GameSimulator,
    home_params: dict,
    away_params: dict,
    context: dict,
    iterations: int = 10_000,
    convergence_threshold: float = 0.005,
    convergence_check_interval: int = 1_000,
    seed: int | None = None,
    common_spreads: list[float] | None = None,
    common_totals: list[float] | None = None,
) -> SimulationOutput:
    """
    Run Monte Carlo simulation and collect distributions.

    Args:
        simulator: Sport-specific game simulator
        home_params: Home team parameters from statistics-service
        away_params: Away team parameters from statistics-service
        context: Game context (venue, surface, dome, neutral site)
        iterations: Maximum number of iterations
        convergence_threshold: Stop early if SE of mean margin < this
        convergence_check_interval: Check convergence every N iterations
        seed: Random seed for reproducibility (None = random)
        common_spreads: Lines to pre-compute cover probabilities for
        common_totals: Totals to pre-compute over probabilities for
    """
    rng = np.random.default_rng(seed)
    simulator.set_parameters(home_params, away_params, context)

    home_scores = np.zeros(iterations, dtype=np.int32)
    away_scores = np.zeros(iterations, dtype=np.int32)
    converged = False
    convergence_iteration = None

    for i in range(iterations):
        result = simulator.simulate_game(rng)
        home_scores[i] = result.home_score
        away_scores[i] = result.away_score

        # Convergence check
        if (i + 1) % convergence_check_interval == 0 and i >= 2_000:
            n = i + 1
            margins_so_far = home_scores[:n] - away_scores[:n]
            se_mean = np.std(margins_so_far, ddof=1) / np.sqrt(n)
            if se_mean < convergence_threshold:
                converged = True
                convergence_iteration = n
                home_scores = home_scores[:n]
                away_scores = away_scores[:n]
                break

    # Compute distributions
    actual_n = len(home_scores)
    margins = home_scores - away_scores
    totals = home_scores + away_scores

    # Pre-compute spread and total probabilities
    spread_covers = {}
    if common_spreads:
        for line in common_spreads:
            # home covers if margin > -line (e.g., home -3.5 covers if margin > 3.5)
            spread_covers[line] = float(np.mean(margins > -line))

    total_overs = {}
    if common_totals:
        for line in common_totals:
            total_overs[line] = float(np.mean(totals > line))

    return SimulationOutput(
        iterations_run=actual_n,
        converged=converged,
        convergence_iteration=convergence_iteration,
        home_scores=home_scores,
        away_scores=away_scores,
        margins=margins,
        totals=totals,
        home_win_prob=float(np.mean(margins > 0)),
        away_win_prob=float(np.mean(margins < 0)),
        draw_prob=float(np.mean(margins == 0)),
        margin_mean=float(np.mean(margins)),
        margin_std=float(np.std(margins, ddof=1)),
        total_mean=float(np.mean(totals)),
        total_std=float(np.std(totals, ddof=1)),
        spread_covers=spread_covers,
        total_overs=total_overs,
        parameters_hash=compute_hash(home_params, away_params, context),
        elapsed_ms=elapsed,
    )
```

### Convergence Diagnostics

The simulation must determine when enough iterations have been run to produce stable estimates. We use two complementary criteria:

**Criterion 1: Standard Error of the Mean (primary)**

The standard error of the mean margin is the primary convergence metric:

```
SE(margin_mean) = std(margins) / sqrt(N)
```

Target: `SE < 0.005` (i.e., the mean margin is estimated to within ~0.01 points with 95% confidence).

For typical football games (margin std ~14), this requires:
- `N = (14 / 0.005)^2 = 7,840,000` -- far too many for strict convergence

In practice, we relax this. The real convergence target is for the **derived probabilities** (win prob, spread cover prob) to be stable, not the mean itself.

**Criterion 2: Probability Stability (practical)**

Check that key probabilities have stabilized between consecutive check intervals:

```
max_prob_change = max(
    |P_new(home_win) - P_prev(home_win)|,
    |P_new(spread_cover) - P_prev(spread_cover)|,  # at the most common line
    |P_new(total_over) - P_prev(total_over)|,       # at the most common line
)
```

Convergence when `max_prob_change < 0.002` (0.2 percentage points) for two consecutive checks.

**Recommended Iteration Counts:**

| Sport | Default Iterations | Convergence Typical | Justification |
|-------|-------------------|-------------------|---------------|
| Football (NFL/NCAA) | 10,000 | 8,000-10,000 | High variance (~14 point margin std), discrete scoring in 3s and 7s creates multimodal distributions that need more samples |
| Basketball (NBA/NCAA) | 10,000 | 5,000-7,000 | Lower variance (~10 point margin std), many possessions per game smooth distributions faster |
| Baseball (MLB/NCAA) | 10,000 | 6,000-8,000 | Low-scoring with occasional blowouts creates right-skewed distributions; moderate iterations needed |
| Any (high-stakes) | 50,000 | 20,000-40,000 | For games with detected edges near the EV threshold, additional precision is warranted |

### Output Format

The `SimulationOutput` maps to bet markets as follows:

| Output Field | Bet Markets Served |
|---|---|
| `home_win_prob`, `away_win_prob` | Moneyline |
| `spread_covers[line]` | Point spread at each line value |
| `total_overs[line]` | Over/under at each total value |
| `margin_mean`, `margin_std` | Spread pricing, alternate spreads |
| `total_mean`, `total_std` | Total pricing, alternate totals |
| `home_scores`, `away_scores` distributions | Team totals, exact score props |
| `margins` distribution | Margin of victory props, alternate spreads |

---

## 2. Football Plugin (NFL + NCAA_FB)

### Simulation Model: Drive-Based

Football is simulated at the **drive level**. Each drive starts at a field position and ends with a terminal outcome (touchdown, field goal, punt, turnover, turnover on downs, safety, end of half/game). Within each drive, plays are simulated to advance the ball and manage down/distance, but the primary modeling unit is the drive outcome conditioned on starting field position and team quality.

### Simulation Loop Pseudocode

```python
class FootballSimulator(GameSimulator):
    """
    Drive-based football simulation for NFL and NCAA_FB.
    """

    def simulate_game(self, rng: np.random.Generator) -> GameResult:
        state = GameState(
            home_score=0, away_score=0,
            quarter=1, time_remaining=900,  # seconds in quarter
            possession='away',  # away receives opening kickoff (configurable)
            field_position=25,  # after kickoff (yards from own goal)
            down=1, distance=10,
            home_timeouts=3, away_timeouts=3,
        )

        while not state.is_game_over():
            # Simulate kickoff/punt return if applicable
            if state.is_kickoff:
                state.field_position = self._simulate_kickoff_return(state, rng)
                state.is_kickoff = False

            # Simulate one drive
            drive_result = self._simulate_drive(state, rng)

            # Apply drive result to game state
            self._apply_drive_result(state, drive_result)

            # Handle scoring (extra point / 2pt conversion)
            if drive_result.outcome == DriveOutcome.TOUCHDOWN:
                self._simulate_pat(state, rng)

            # Switch possession
            state.switch_possession()

            # Set starting field position for next drive
            if drive_result.outcome in (DriveOutcome.PUNT, DriveOutcome.TOUCHBACK):
                state.field_position = self._simulate_punt(state, rng)
            elif drive_result.outcome == DriveOutcome.FIELD_GOAL:
                state.field_position = 25  # kickoff
                state.is_kickoff = True
            elif drive_result.outcome == DriveOutcome.TOUCHDOWN:
                state.field_position = 25  # kickoff
                state.is_kickoff = True
            elif drive_result.outcome in (DriveOutcome.TURNOVER, DriveOutcome.TURNOVER_ON_DOWNS):
                state.field_position = 100 - drive_result.end_field_position

        return GameResult(
            home_score=state.home_score,
            away_score=state.away_score,
            metadata={'quarters': state.quarter}
        )

    def _simulate_drive(self, state: GameState, rng) -> DriveResult:
        """
        Simulate a single offensive drive.

        Each play within the drive updates down, distance, and field position.
        The drive ends when a terminal event occurs.
        """
        plays = 0
        while True:
            plays += 1

            # Determine play outcome based on game state
            play = self._simulate_play(state, rng)

            # Check for turnover
            if play.is_turnover:
                return DriveResult(
                    outcome=DriveOutcome.TURNOVER,
                    end_field_position=state.field_position,
                    plays=plays,
                    time_elapsed=plays * self._avg_play_time(state),
                )

            # Advance ball
            state.field_position += play.yards_gained

            # Check for touchdown
            if state.field_position >= 100:
                return DriveResult(
                    outcome=DriveOutcome.TOUCHDOWN,
                    end_field_position=100,
                    plays=plays,
                    time_elapsed=plays * self._avg_play_time(state),
                )

            # Check for safety (pushed back past own goal line)
            if state.field_position <= 0:
                return DriveResult(
                    outcome=DriveOutcome.SAFETY,
                    end_field_position=0,
                    plays=plays,
                    time_elapsed=plays * self._avg_play_time(state),
                )

            # Update down and distance
            if play.yards_gained >= state.distance:
                state.down = 1
                state.distance = min(10, 100 - state.field_position)
            else:
                state.distance -= play.yards_gained
                state.down += 1

            # Fourth down decision
            if state.down > 4:
                return DriveResult(
                    outcome=DriveOutcome.TURNOVER_ON_DOWNS,
                    end_field_position=state.field_position,
                    plays=plays,
                    time_elapsed=plays * self._avg_play_time(state),
                )

            if state.down == 4:
                decision = self._fourth_down_decision(state)
                if decision == FourthDownDecision.PUNT:
                    return DriveResult(
                        outcome=DriveOutcome.PUNT,
                        end_field_position=state.field_position,
                        plays=plays,
                        time_elapsed=plays * self._avg_play_time(state),
                    )
                elif decision == FourthDownDecision.FIELD_GOAL:
                    if self._attempt_field_goal(state, rng):
                        return DriveResult(
                            outcome=DriveOutcome.FIELD_GOAL,
                            end_field_position=state.field_position,
                            plays=plays,
                            time_elapsed=plays * self._avg_play_time(state),
                        )
                    else:
                        return DriveResult(
                            outcome=DriveOutcome.MISSED_FG,
                            end_field_position=state.field_position,
                            plays=plays,
                            time_elapsed=plays * self._avg_play_time(state),
                        )
                # else: GO_FOR_IT -- continue the loop with 4th down play
```

### Input Parameters and Sources

| Parameter | Source (statistics-service endpoint) | Description |
|---|---|---|
| `off_epa_per_play` | `GET /teams/{id}/stats?category=offense` | Offensive EPA per play (overall and by down) |
| `def_epa_per_play` | `GET /teams/{id}/stats?category=defense` | Defensive EPA per play (overall and by down) |
| `yards_per_play_dist` | `GET /teams/{id}/stats?category=offense` | Distribution of yards gained per play (mean, std, skew) |
| `turnover_rate` | `GET /teams/{id}/stats?category=offense` | Turnovers per play (fumbles + interceptions) |
| `forced_turnover_rate` | `GET /teams/{id}/stats?category=defense` | Opponent turnovers forced per play |
| `third_down_conv_rate` | `GET /teams/{id}/stats?category=offense` | Third-down conversion percentage |
| `red_zone_td_rate` | `GET /teams/{id}/stats?category=offense` | TD rate when inside the 20 |
| `red_zone_fg_rate` | `GET /teams/{id}/stats?category=offense` | FG rate when inside the 20 |
| `fourth_down_go_rate` | `GET /teams/{id}/stats?category=offense` | Rate of going for it on 4th down (by distance and field position) |
| `fg_make_pct_by_dist` | `GET /teams/{id}/stats?category=special_teams` | FG make probability by distance bucket (20-29, 30-39, 40-49, 50+) |
| `punt_distance_dist` | `GET /teams/{id}/stats?category=special_teams` | Punt distance distribution (mean, std) |
| `kick_return_dist` | `GET /teams/{id}/stats?category=special_teams` | Kick return yardage distribution |
| `plays_per_game` | `GET /teams/{id}/stats?category=offense` | Pace/tempo proxy |
| `pass_play_pct` | `GET /teams/{id}/stats?category=offense` | Pass vs. run ratio (adjusts by game state) |
| `two_pt_conv_rate` | `GET /teams/{id}/stats?category=special_teams` | Two-point conversion success rate |
| `venue_info` | `GET /venues/{id}` | Dome/outdoor, surface type, altitude |

### Play Outcome Model

Each play outcome is drawn from a distribution conditioned on down, distance, field position, and team quality. The yards-gained distribution is modeled as a **mixture of normals** to capture the multimodal nature of football plays:

```
P(yards | down, distance, field_pos) =
    w_neg * Normal(mu_neg, sigma_neg)     # negative plays (sacks, TFLs)
  + w_short * Normal(mu_short, sigma_short) # short gains (0-5 yards)
  + w_med * Normal(mu_med, sigma_med)       # medium gains (5-15 yards)
  + w_long * Exponential(lambda_long)       # explosive plays (15+ yards)
```

Weights and parameters are derived from team offensive/defensive efficiency stats.

**Turnover probability per play:**

```
P(turnover) = (off_turnover_rate + def_forced_turnover_rate) / 2
```

Adjusted by field position (turnovers are slightly more likely in a team's own territory due to desperation passing) and down/distance (long-yardage situations have higher interception probability).

### Score Transitions

| Event | Points | Conditions |
|---|---|---|
| Touchdown + PAT | 7 | PAT make rate: ~94% (NFL), ~93% (NCAA) |
| Touchdown + 2pt | 8 | 2pt conversion rate: ~48% (NFL), ~45% (NCAA) |
| Touchdown + failed PAT/2pt | 6 | Complement of above |
| Field Goal | 3 | Make probability by distance (see table below) |
| Safety | 2 | Ball carrier tackled in own end zone |

**Two-point conversion decision model:** Teams attempt 2pt conversions based on score differential and game clock. The simulation uses a lookup table based on common coaching decision charts (e.g., trailing by 5, go for 2 after a TD to take a 1-point lead).

### Field Goal Probability Model

FG probability is modeled as a logistic function of distance:

```
P(make | distance) = 1 / (1 + exp(-(beta_0 + beta_1 * distance)))
```

Baseline NFL parameters (from historical data):
- `beta_0 = 5.5`
- `beta_1 = -0.105`

This yields approximately:
| Distance | P(make) NFL | P(make) NCAA |
|----------|-------------|--------------|
| 20-29 yds | 97% | 95% |
| 30-39 yds | 90% | 85% |
| 40-49 yds | 78% | 70% |
| 50-59 yds | 58% | 45% |
| 60+ yds | 30% | 15% |

NCAA values are lower due to less consistent kicking talent. Team-specific kicker quality adjusts `beta_0` by up to +/-0.5.

### Special Teams Models

**Punt distance:**

```
punt_distance ~ Normal(mu=45, sigma=5)  # NFL
punt_distance ~ Normal(mu=42, sigma=6)  # NCAA
```

Net punt distance accounts for return yardage:

```
return_yards ~ max(0, Normal(mu=8, sigma=6))  # most punts are fair-caught or short returns
P(touchback) = 0.10  # fair catch inside 10 or touchback
P(muffed_punt) = 0.01
```

**Kickoff model:**

```
P(touchback) = 0.55  # NFL (since rule changes)
P(touchback) = 0.45  # NCAA
return_yards | not_touchback ~ max(0, Normal(mu=23, sigma=8))
P(kick_return_td) = 0.005
```

### NCAA Football Differences

The simulation uses a `league` flag to swap parameters:

| Aspect | NFL | NCAA_FB |
|---|---|---|
| Overtime rules | Sudden death (modified 2024) | Alternating possessions from 25-yd line; 2pt required after 2nd OT |
| Clock on first down | Continuous | Stops momentarily (conference-dependent) |
| Variance multiplier | 1.0x | 1.3x (wider talent gaps, less scheme consistency) |
| Yards per play dist | Narrower | Wider tails (more explosive plays AND more negative plays) |
| Fourth-down aggression | Moderate (team-specific) | Varies wildly by coaching staff |
| Data quality adjustment | None | Apply Bayesian shrinkage toward conference-level priors for small-sample teams |

**Bayesian parameter shrinkage for NCAA:**

For teams with fewer than 8 games of data (early season), parameters are shrunk toward conference-average priors:

```
param_adjusted = (n * team_param + k * conference_prior) / (n + k)
```

Where `k` is the shrinkage strength (recommend `k = 4`, meaning 4 games of prior weight). This prevents overreacting to early-season results against weak opponents.

### Key Simplifying Assumptions

1. **Plays within a drive are i.i.d. conditioned on down/distance/field position.** In reality, play-calling tendencies create autocorrelation (a run on 1st down shifts the distribution for 2nd down). Impact: minor -- the down/distance conditioning captures most of this.

2. **No momentum modeling.** Scoring drives do not make subsequent drives more likely to score. The empirical evidence for momentum in football is weak, so this is a reasonable simplification.

3. **Clock management is approximate.** We estimate time per play rather than tracking the play clock precisely. Impact: small for final score prediction, moderate for live/in-game modeling.

4. **Defensive and offensive parameters are independent.** A matchup between a strong offense and strong defense uses the average of their respective parameters. In reality, specific scheme matchups matter (e.g., a rush-heavy offense vs. a pass-defense-focused unit). Impact: moderate -- the ML adjustment layer is designed to capture these matchup effects.

5. **No individual player modeling.** The simulation uses team-level parameters. A quarterback injury would require re-parameterization. Impact: significant for specific scenarios, but handled by the prediction-engine's injury adjustment.

### Computational Cost Estimate

- **Plays per simulated game:** ~130 (NFL), ~140 (NCAA, higher tempo variance)
- **Time per simulated game:** ~0.05ms (with NumPy vectorization of play outcomes)
- **10,000 iterations:** ~500ms per game
- **50,000 iterations:** ~2.5s per game
- **Full daily batch (15 NFL games):** ~7.5s at 10K iterations
- **Full weekly NCAA batch (60 FBS games):** ~30s at 10K iterations

These estimates assume a single-threaded Python process. Parallelization across games provides linear speedup.

---

## 3. Basketball Plugin (NBA + NCAA_BB)

### Simulation Model: Possession-Based

Basketball is simulated at the **possession level**. Each possession results in a terminal outcome: made field goal (2pt or 3pt), free throws, turnover, or offensive rebound (which extends the possession). The simulation alternates possessions between teams, tracking score, fouls, and time.

### Simulation Loop Pseudocode

```python
class BasketballSimulator(GameSimulator):
    """
    Possession-based basketball simulation for NBA and NCAA_BB.
    """

    def simulate_game(self, rng: np.random.Generator) -> GameResult:
        state = GameState(
            home_score=0, away_score=0,
            period=1,
            time_remaining=self._period_length(),  # 720s (NBA quarter) or 1200s (NCAA half)
            possession='away',  # tip-off winner (randomized ~50/50)
            home_team_fouls=0, away_team_fouls=0,
            home_player_fouls=[0]*5, away_player_fouls=[0]*5,  # simplified: track 5 starters
            periods_total=self._num_periods(),  # 4 (NBA) or 2 (NCAA)
        )

        # Determine game pace
        # Pace interaction: average of the two teams' paces, regression to league mean
        game_pace = self._calculate_game_pace()
        possession_duration = self._period_length() * self._num_periods() / game_pace

        while not state.is_game_over():
            # Simulate one possession
            poss_result = self._simulate_possession(state, rng)

            # Apply scoring
            self._apply_possession_result(state, poss_result)

            # Advance clock
            poss_time = rng.exponential(possession_duration)
            poss_time = np.clip(poss_time, 5, self._shot_clock())
            state.time_remaining -= poss_time

            # Check for period end
            if state.time_remaining <= 0:
                self._advance_period(state)

            # Switch possession (unless offensive rebound)
            if not poss_result.offensive_rebound:
                state.switch_possession()

        # Handle overtime if tied at end of regulation
        while state.home_score == state.away_score:
            state.start_overtime()
            self._simulate_overtime(state, rng, game_pace)

        return GameResult(
            home_score=state.home_score,
            away_score=state.away_score,
            metadata={'periods': state.period, 'overtime': state.period > self._num_periods()}
        )

    def _simulate_possession(self, state: GameState, rng) -> PossessionResult:
        """
        Simulate a single offensive possession.
        Outcome tree:
          1. Turnover? -> no points, possession change
          2. Shot attempt:
             a. Free throws (foul on shot)?
             b. 3pt attempt or 2pt attempt?
             c. Made or missed?
             d. If missed: offensive rebound? -> repeat possession
             e. And-1 foul? -> bonus free throw
        """
        off_params = self._get_offense_params(state)
        def_params = self._get_defense_params(state)

        # --- Turnover check ---
        # Turnover rate: average of offense's TO rate and defense's forced TO rate
        to_rate = (off_params['tov_pct'] + def_params['forced_tov_pct']) / 2.0 / 100.0
        if rng.random() < to_rate:
            return PossessionResult(points=0, offensive_rebound=False, foul=False)

        # --- Shot selection ---
        # Probability of 3pt attempt vs 2pt attempt
        three_rate = off_params['three_attempt_rate']
        is_three = rng.random() < three_rate

        # --- Foul on shot (shooting foul before/during attempt) ---
        ft_rate = (off_params['ft_rate'] + def_params['opp_ft_rate']) / 2.0
        P_shooting_foul = ft_rate * 0.15  # ~15% of possessions result in FTs
        if rng.random() < P_shooting_foul:
            self._apply_team_foul(state)
            n_fts = 3 if is_three else 2
            ft_points = sum(1 for _ in range(n_fts) if rng.random() < off_params['ft_pct'])
            return PossessionResult(points=ft_points, offensive_rebound=False, foul=True)

        # --- Shot attempt ---
        if is_three:
            make_prob = (off_params['three_pct'] + (1.0 - def_params['opp_three_pct'])) / 2.0
            # Blend offense and defense: average offensive 3P% and inverse of defensive 3P% allowed
            # Simplified: use harmonic blend
            make_prob = 2.0 * off_params['three_pct'] * (1.0 - def_params['opp_three_pct_adj']) / \
                        (off_params['three_pct'] + (1.0 - def_params['opp_three_pct_adj']))
            # Fallback to simpler model:
            make_prob = (off_params['three_pct'] * 0.5) + ((1 - def_params['opp_three_pct_from_lg']) * 0.5)
        else:
            make_prob = off_params['two_pct_adj']
            # Adjust for defense
            make_prob = (off_params['two_pct'] + (1 - def_params['opp_two_pct_adj'])) / 2.0

        if rng.random() < make_prob:
            points = 3 if is_three else 2

            # And-1 check (~3% of made baskets)
            if not is_three and rng.random() < 0.03:
                self._apply_team_foul(state)
                if rng.random() < off_params['ft_pct']:
                    points += 1

            return PossessionResult(points=points, offensive_rebound=False, foul=False)

        # --- Miss: check for offensive rebound ---
        oreb_rate = (off_params['oreb_pct'] + def_params['opp_oreb_pct']) / 2.0 / 100.0
        if rng.random() < oreb_rate:
            return PossessionResult(points=0, offensive_rebound=True, foul=False)

        return PossessionResult(points=0, offensive_rebound=False, foul=False)
```

### Input Parameters and Sources

| Parameter | Source (statistics-service endpoint) | Description |
|---|---|---|
| `off_rating` | `GET /teams/{id}/stats?category=offense` | Points per 100 possessions |
| `def_rating` | `GET /teams/{id}/stats?category=defense` | Points allowed per 100 possessions |
| `pace` | `GET /teams/{id}/stats?category=general` | Possessions per game (NBA: ~48min, NCAA: ~40min) |
| `efg_pct` | `GET /teams/{id}/stats?category=offense` | Effective field goal percentage |
| `three_pct` | `GET /teams/{id}/stats?category=offense` | Three-point field goal percentage |
| `three_attempt_rate` | `GET /teams/{id}/stats?category=offense` | 3PA / FGA ratio |
| `two_pct` | `GET /teams/{id}/stats?category=offense` | Two-point field goal percentage |
| `ft_pct` | `GET /teams/{id}/stats?category=offense` | Free throw percentage |
| `ft_rate` | `GET /teams/{id}/stats?category=offense` | FTA / FGA ratio |
| `tov_pct` | `GET /teams/{id}/stats?category=offense` | Turnover percentage (turnovers per 100 possessions) |
| `oreb_pct` | `GET /teams/{id}/stats?category=offense` | Offensive rebound percentage |
| `opp_three_pct` | `GET /teams/{id}/stats?category=defense` | Opponent 3P% allowed |
| `opp_two_pct` | `GET /teams/{id}/stats?category=defense` | Opponent 2P% allowed |
| `opp_ft_rate` | `GET /teams/{id}/stats?category=defense` | Opponent FT rate allowed |
| `forced_tov_pct` | `GET /teams/{id}/stats?category=defense` | Forced turnovers per 100 possessions |
| `opp_oreb_pct` | `GET /teams/{id}/stats?category=defense` | Opponent offensive rebound % allowed |

### Pace Interaction Model

When two teams with different paces meet, the resulting game pace is not a simple average. Empirically, the slower team has more control over the pace. We use a weighted average with regression to the league mean:

```
game_pace = 0.5 * (home_pace + away_pace) * 0.6 + league_avg_pace * 0.4
```

Refinement: weight the slower team more heavily:

```
game_pace = (min(home_pace, away_pace) * 0.55 + max(home_pace, away_pace) * 0.45)
            * 0.7 + league_avg_pace * 0.3
```

This matters greatly for totals: a game between a team that plays at 75 possessions/game and one at 65 possessions/game will not play at 70 -- the slower team drags the pace below the midpoint.

### Foul Accumulation Model

Fouls are tracked at the team level (for bonus/double-bonus) and simplified player level (for disqualification risk). The foul model:

```
P(foul_on_possession) = base_foul_rate * (1 + aggression_adjustment)
```

Where `aggression_adjustment` increases when the defending team is trailing and decreases when leading (trailing teams play more aggressively on defense).

**Bonus thresholds:**
- **NBA:** Bonus at 5th team foul per quarter; player fouls out at 6
- **NCAA:** Bonus at 7th team foul per half (1-and-1); double bonus at 10th (2 FTs); player fouls out at 5

When a key player accumulates fouls (3 in the first half, 4 in the second half), the simulation reduces that team's offensive and defensive ratings to reflect bench-player substitution:

```
rating_adjustment = -1.5 * n_starters_in_foul_trouble  # points per 100 possessions
```

### Output Distributions and Market Mapping

| Distribution | Markets |
|---|---|
| Final score pairs (home, away) | Moneyline, exact score |
| Margin distribution | Spread, alternate spreads |
| Total distribution | Over/under, alternate totals |
| Quarter-by-quarter scoring (if tracked) | First-half spread/total, quarter props |
| Individual scoring (if lineup-level sim) | Player points props |

### NCAA Basketball Differences

| Aspect | NBA | NCAA_BB |
|---|---|---|
| Game length | 48 min (4 x 12) | 40 min (2 x 20) |
| Shot clock | 24 seconds | 30 seconds |
| Three-point distance | 23'9" | 22'1.75" |
| Foul limit (player) | 6 | 5 |
| Bonus threshold | 5th per quarter | 7th per half (1-and-1), 10th (double) |
| Possessions per game | ~100 per team | ~65-70 per team |
| Pace variance | Moderate (90-105) | High (60-75) |
| Talent variance | Low (parity) | Very high (top 10 vs. mid-major) |
| Data quality | Excellent | Moderate |

**NCAA-specific adjustments:**

1. **Higher three-point percentage** due to shorter line: NCAA 3P% averages ~34% vs. NBA ~36%, but the shorter distance means shot selection dynamics differ. NCAA teams shoot more mid-range relative to the NBA.

2. **Wider outcome distributions:** Apply a variance multiplier of 1.2x to the margins for NCAA games to reflect greater unpredictability.

3. **Bayesian shrinkage** for teams with sparse data:
   ```
   param = (n_games * team_param + k * conference_prior) / (n_games + k)
   ```
   With `k = 5` (5 games of prior weight). Conference priors are computed from the aggregate of teams in the same conference.

4. **Zone defense adjustment:** NCAA teams use zone defense more frequently. Teams facing zone defenses shoot fewer threes and have higher turnover rates. If the defending team is a known zone team (flagged in stats), adjust:
   ```
   off_tov_pct += 1.5  # zone creates more turnovers
   off_three_attempt_rate -= 0.03  # fewer three-point attempts
   ```

### Key Simplifying Assumptions

1. **Team-level parameters, not lineup-level.** The simulation uses aggregate team ratings rather than tracking which 5 players are on the floor. Impact: moderate -- lineup effects are real in basketball, but lineup-level simulation adds substantial complexity and the ML layer can adjust for known lineup changes (injuries, rest).

2. **Possessions are i.i.d. within a game.** No momentum or hot-hand modeling. Impact: small -- the empirical evidence for basketball momentum is debatable, and the hot-hand effect, while real, is small (~1.5 percentage point shooting boost).

3. **No garbage time adjustment in simulation.** When a game becomes a blowout, the simulation does not change parameters. Impact: small for final score prediction (garbage time affects who scores but not the total much), moderate for margin/spread accuracy.

4. **Overtime is modeled as a mini-game** with the same parameters. Impact: minimal -- overtime games are ~6% of NBA games and the parameters do not change meaningfully.

### Computational Cost Estimate

- **Possessions per simulated game:** ~200 (NBA), ~135 (NCAA)
- **Time per simulated game:** ~0.02ms
- **10,000 iterations:** ~200ms per game
- **50,000 iterations:** ~1s per game
- **Full daily batch (15 NBA games):** ~3s at 10K iterations
- **Full daily batch (50 NCAA_BB games during conference play):** ~10s at 10K iterations

---

## 4. Baseball Plugin (MLB + NCAA_BSB)

### Simulation Model: Plate-Appearance Based

Baseball is simulated at the **plate appearance** level. Each PA produces a discrete outcome (single, double, triple, home run, walk, HBP, strikeout, ground out, fly out, etc.). The game state is tracked as a Markov chain over 24 states (8 baserunner configurations x 3 out counts) per half-inning, with the batting order cycling through 9 hitters.

### Simulation Loop Pseudocode

```python
class BaseballSimulator(GameSimulator):
    """
    Plate-appearance simulation for MLB and NCAA_BSB.
    """

    def simulate_game(self, rng: np.random.Generator) -> GameResult:
        state = GameState(
            home_score=0, away_score=0,
            inning=1, half='top',  # top = away batting, bottom = home batting
            outs=0,
            bases=[False, False, False],  # first, second, third
            away_lineup_pos=0,  # 0-8
            home_lineup_pos=0,
            away_pitcher=self.away_starter,
            home_pitcher=self.home_starter,
            away_pitch_count=0,
            home_pitch_count=0,
        )

        max_innings = 9  # standard (7 for NCAA doubleheaders)

        while not state.is_game_over(max_innings):
            # Get current batter and pitcher
            batter = self._get_current_batter(state)
            pitcher = self._get_current_pitcher(state)

            # Check for pitching change
            if self._should_change_pitcher(state, pitcher):
                pitcher = self._select_reliever(state, rng)
                self._set_pitcher(state, pitcher)

            # Check for pinch hitter (late game, pitcher spot in NL-era data)
            if self._should_pinch_hit(state, batter, pitcher):
                batter = self._select_pinch_hitter(state)

            # Simulate plate appearance
            outcome = self._simulate_pa(batter, pitcher, state, rng)

            # Apply outcome to game state
            self._apply_pa_outcome(state, outcome)

            # Advance lineup position
            self._advance_lineup(state)

            # Check for 3 outs
            if state.outs >= 3:
                self._end_half_inning(state)

            # Check mercy rule (NCAA: 10-run lead after 7 innings)
            if self.league == 'NCAA_BSB' and self._mercy_rule_applies(state):
                break

        return GameResult(
            home_score=state.home_score,
            away_score=state.away_score,
            metadata={
                'innings': state.inning,
                'home_hits': state.home_hits,
                'away_hits': state.away_hits,
            }
        )

    def _simulate_pa(self, batter: BatterParams, pitcher: PitcherParams,
                     state: GameState, rng) -> PAOutcome:
        """
        Simulate a single plate appearance using the log5 method
        to combine batter and pitcher talent.

        The log5 method for estimating matchup probability:
            P(event) = (P_batter * P_pitcher / P_league) /
                       (P_batter * P_pitcher / P_league +
                        (1 - P_batter) * (1 - P_pitcher) / (1 - P_league))
        """
        # Step 1: Determine walk/HBP
        bb_rate = self._log5(batter.bb_pct, pitcher.bb_pct, self.league_bb_pct)
        hbp_rate = self._log5(batter.hbp_pct, pitcher.hbp_pct, self.league_hbp_pct)

        r = rng.random()
        if r < bb_rate:
            return PAOutcome.WALK
        r -= bb_rate
        if r < hbp_rate:
            return PAOutcome.HBP

        # Step 2: Determine strikeout
        k_rate = self._log5(batter.k_pct, pitcher.k_pct, self.league_k_pct)
        # Adjust remaining probability space (conditional on ball in play or K)
        remaining = 1.0 - bb_rate - hbp_rate
        k_prob_adj = k_rate / remaining if remaining > 0 else 0

        if rng.random() < k_prob_adj:
            return PAOutcome.STRIKEOUT

        # Step 3: Ball in play -- determine outcome
        # BABIP (batting average on balls in play)
        babip = self._log5(batter.babip, pitcher.babip, self.league_babip)

        # Apply park factor
        babip *= self.park_factors['babip']

        if rng.random() < babip:
            # Hit -- determine type
            return self._determine_hit_type(batter, pitcher, state, rng)
        else:
            # Out -- determine type (ground out, fly out, line out)
            return self._determine_out_type(batter, pitcher, state, rng)

    def _determine_hit_type(self, batter, pitcher, state, rng) -> PAOutcome:
        """
        Given a hit, determine single/double/triple/HR.
        Uses batter's ISO (isolated power) and HR/FB rate,
        adjusted by pitcher and park factors.
        """
        hr_rate = self._log5(batter.hr_fb_rate, pitcher.hr_fb_rate, self.league_hr_fb_rate)
        hr_rate *= self.park_factors['hr']

        # Apply TTOP (times through the order penalty)
        times_through = self._times_through_order(state)
        if times_through >= 2:
            # Pitcher gets worse: ~0.010 wOBA per time through
            hr_rate *= (1 + 0.05 * (times_through - 1))

        # Probabilities conditional on hit
        P_hr = hr_rate * 0.35  # ~35% of hits are fly balls, HR/FB converts some to HR
        P_triple = 0.02  # ~2% of hits are triples
        P_double = 0.18  # ~18% of hits are doubles
        P_single = 1.0 - P_hr - P_triple - P_double

        # Adjust by park
        P_double *= self.park_factors['2b']
        P_triple *= self.park_factors['3b']

        # Renormalize
        total = P_single + P_double + P_triple + P_hr
        r = rng.random() * total

        if r < P_hr:
            return PAOutcome.HOME_RUN
        r -= P_hr
        if r < P_triple:
            return PAOutcome.TRIPLE
        r -= P_triple
        if r < P_double:
            return PAOutcome.DOUBLE
        return PAOutcome.SINGLE

    def _log5(self, p_batter: float, p_pitcher: float, p_league: float) -> float:
        """
        Log5 method: combine batter and pitcher probabilities
        relative to the league average.

        Formula:
            p = (p_b * p_p / p_l) /
                (p_b * p_p / p_l + (1-p_b) * (1-p_p) / (1-p_l))
        """
        num = p_batter * p_pitcher / p_league
        denom = num + (1 - p_batter) * (1 - p_pitcher) / (1 - p_league)
        return num / denom if denom > 0 else p_league
```

### Baserunner Advancement Model

When a hit occurs, baserunners advance according to a probabilistic model:

| Event | Runner on 1st | Runner on 2nd | Runner on 3rd |
|---|---|---|---|
| Single | Advances to 2nd (80%), to 3rd (20%) | Scores (60%), to 3rd (40%) | Scores (95%) |
| Double | Scores (40%), to 3rd (60%) | Scores (95%) | Scores (99%) |
| Triple | Scores (99%) | Scores (99%) | Scores (99%) |
| Home Run | Scores | Scores | Scores |
| Walk/HBP | Forced advance only | Forced if runner on 1st | Forced if runners on 1st+2nd |
| Ground out | Advance 1 base (varies) | Advance (70%) | Scores on sac fly analogue |
| Fly out (deep) | Hold (90%) | Tag up to 3rd (40%) | Scores (sacrifice fly, 50%) |

These probabilities are simplified league averages. The simulation does not model individual baserunner speed (a simplifying assumption).

### Input Parameters and Sources

| Parameter | Source | Description |
|---|---|---|
| `lineup[0..8]` | `GET /games/{id}` (lineups endpoint) | Batting order with player IDs |
| `batter.ba`, `obp`, `slg` | `GET /players/{id}/stats` | Traditional batting stats |
| `batter.k_pct`, `bb_pct` | `GET /players/{id}/stats` | Plate discipline rates |
| `batter.babip` | `GET /players/{id}/stats` | Batting avg on balls in play |
| `batter.hr_fb_rate` | `GET /players/{id}/stats` | Home run per fly ball rate |
| `batter.iso` | `GET /players/{id}/stats` | Isolated power (SLG - BA) |
| `pitcher.k_pct`, `bb_pct` | `GET /players/{id}/stats` | Pitcher K and BB rates |
| `pitcher.babip` | `GET /players/{id}/stats` | Pitcher BABIP allowed |
| `pitcher.hr_fb_rate` | `GET /players/{id}/stats` | HR/FB allowed |
| `pitcher.fip`, `xfip` | `GET /players/{id}/stats` | Fielding-independent pitching |
| `pitcher.pitch_count_limit` | `GET /players/{id}/stats` | Expected pitch count threshold |
| `bullpen[]` | `GET /teams/{id}/stats?category=bullpen` | Reliever stats and availability |
| `park_factors` | `GET /venues/{id}` | HR, 2B, 3B, run park factors |
| `league_avg_*` | `GET /leagues/{id}/stats` | League average rates for log5 |
| `platoon_splits` | `GET /players/{id}/stats?split=platoon` | L/R splits for batter and pitcher |

### Pitching Change Model

The simulation decides when to pull the starting pitcher based on:

```python
def _should_change_pitcher(self, state, pitcher) -> bool:
    # Rule 1: Pitch count
    if state.pitch_count >= pitcher.pitch_count_limit:
        return True

    # Rule 2: Times through order penalty (TTOP)
    times_through = self._times_through_order(state)
    if times_through >= 3:
        # High probability of being pulled after 3rd time through
        return self.rng.random() < 0.70

    # Rule 3: Performance degradation (high pitch count even if under limit)
    if state.pitch_count > 85:
        degradation_prob = (state.pitch_count - 85) * 0.02  # 2% per pitch over 85
        return self.rng.random() < degradation_prob

    # Rule 4: Inning and score (protect leads late)
    if state.inning >= 7 and self._team_is_leading(state) and self._lead_is_small(state):
        return self.rng.random() < 0.50

    return False
```

**TTOP (Times Through the Order Penalty):**

A well-documented effect where starting pitchers become less effective each time they face the batting order:

```
wOBA_adjustment = 0.010 * max(0, times_through - 1)
```

The simulation applies this by increasing the batter's effective stats:

```
batter_adj_babip = batter.babip + 0.008 * max(0, times_through - 1)
batter_adj_hr_rate = batter.hr_fb_rate * (1 + 0.04 * max(0, times_through - 1))
```

### Park Factors

Park factors are multiplicative adjustments applied to batted-ball outcome probabilities. They are sourced from the statistics-service `GET /venues/{venue_id}` endpoint.

| Park Factor | What It Adjusts | Example Values |
|---|---|---|
| `run_factor` | Overall run environment | Coors: 1.30, Oracle Park: 0.90 |
| `hr_factor` | Home run probability | Yankee Stadium: 1.15, Coors: 1.30 |
| `2b_factor` | Double probability | Fenway: 1.25 (Green Monster), Coors: 1.10 |
| `3b_factor` | Triple probability | Coors: 1.40, most parks: 0.90-1.10 |
| `babip_factor` | BABIP adjustment | Coors: 1.05, Tropicana: 0.97 |

Park factors are applied as:

```
adjusted_rate = base_rate * park_factor
```

**Temperature and altitude adjustment (for outdoor parks):**

```
hr_distance_adjustment = 1.0 + (temperature_f - 72) * 0.002 + (altitude_ft / 1000) * 0.01
```

At Coors Field (5,280 ft, 75F): `1.0 + 0.006 + 0.053 = 1.059` additional HR factor beyond the park factor.

### Platoon Split Handling

The simulation uses handedness-specific stats when available. The matchup between batter handedness and pitcher handedness is the most significant individual-level adjustment:

```
if batter.bats == 'L' and pitcher.throws == 'R':
    use batter.vs_rhp stats, pitcher.vs_lhb stats
elif batter.bats == 'R' and pitcher.throws == 'L':
    use batter.vs_lhp stats, pitcher.vs_rhb stats
# ... etc.
```

League-average platoon splits (for Bayesian priors when sample is small):

| Matchup | BA Advantage | OBP Advantage |
|---|---|---|
| RHB vs LHP | +0.015 | +0.020 |
| LHB vs RHP | +0.012 | +0.018 |

### NCAA Baseball Differences

| Aspect | MLB | NCAA_BSB |
|---|---|---|
| Bat material | Wood | Aluminum (BBCOR-certified) |
| Game length | 9 innings always | 9 innings (7 in doubleheaders) |
| Mercy rule | None | 10-run rule after 7 innings (conference-dependent) |
| DH | Universal (since 2022) | Universal |
| Avg team BA | .245-.250 | .270-.290 |
| Avg runs/game | 4.0-4.5 per team | 5.0-6.0 per team |
| K rate | ~22% | ~18% (aluminum bat = larger sweet spot) |
| HR/game | ~1.1 per team | ~0.7 per team (despite aluminum; BBCOR limits exit velo) |
| Pitching depth | 5-man rotation + deep bullpen | 3-man weekend rotation + thin bullpen |
| Data quality | Excellent (Statcast) | Poor (box scores only for many teams) |

**Aluminum bat calibration:** The simulation adjusts outcome probabilities to reflect the aluminum bat environment:

```python
if self.league == 'NCAA_BSB':
    # Aluminum bat adjustments (relative to MLB baseline)
    babip_adj = 1.08       # higher BABIP due to larger sweet spot
    k_rate_adj = 0.85      # fewer strikeouts
    hr_rate_adj = 0.70     # fewer HR despite aluminum (BBCOR limits)
    double_rate_adj = 1.15 # more doubles (harder hit balls, fewer HR)
    total_runs_adj = 1.25  # ~25% more runs per game than MLB equivalent talent
```

**Weekend series pitching model:** College baseball uses a predictable Friday/Saturday/Sunday starter deployment:

```python
if day_of_week == 'FRIDAY':
    starter_quality = team.pitchers[0]  # Ace
elif day_of_week == 'SATURDAY':
    starter_quality = team.pitchers[1]  # #2 starter
elif day_of_week == 'SUNDAY':
    starter_quality = team.pitchers[2]  # #3 starter
else:  # midweek
    starter_quality = team.pitchers[3]  # #4 starter, often significantly worse
```

**Data quality fallback:** For NCAA teams with insufficient individual player data, the simulation falls back to team-level parameterization:

```python
if player_data_available:
    # Use individual batter/pitcher log5 matchup model
    outcome = self._simulate_pa_individual(batter, pitcher, state, rng)
else:
    # Fall back to team-level outcome distributions
    # Team batting line -> expected PA outcome probabilities
    outcome = self._simulate_pa_team_level(off_team, def_team, state, rng)
```

### Key Simplifying Assumptions

1. **Baserunner advancement is probabilistic, not player-specific.** Fast runners would advance more aggressively. Impact: small for run totals (~0.1 runs/game), moderate for specific baserunner state probabilities.

2. **Bullpen usage follows a simplified decision tree.** Real managers consider handedness matchups, recent workload, and game leverage more precisely. Impact: moderate for individual inning scoring, small for game totals since the overall bullpen quality is correct.

3. **No defensive positioning or shift modeling.** Impact: minimal post-2023 (shift ban) for MLB; moderate for NCAA where defensive quality is more variable and errors are more common.

4. **Stolen bases are not explicitly modeled.** They are absorbed into the baserunner advancement probabilities. Impact: small (stolen bases contribute ~0.1-0.2 runs/game).

5. **Errors are implicitly captured via BABIP.** The simulation does not separately model defensive errors. Impact: small for MLB, moderate for NCAA where error rates are higher (~2x MLB rates).

### Computational Cost Estimate

- **Plate appearances per simulated game:** ~75 (MLB), ~80 (NCAA, more scoring)
- **Time per simulated game:** ~0.03ms
- **10,000 iterations:** ~300ms per game
- **50,000 iterations:** ~1.5s per game
- **Full daily batch (15 MLB games):** ~4.5s at 10K iterations
- **Full weekly NCAA batch (100 games):** ~30s at 10K iterations
