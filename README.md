# MultiPeriodConvex — Dual-Sleeve Alpha Engine for Indian Equities

> **KRITI 2026 Quantitative Finance Challenge · Hostel 2613**
> Long-Only Alpha Modeling in Equity Markets

---

## Overview

This repository contains a fully systematic, rules-based equity trading strategy built for the Indian equity market (NSE). The architecture combines two uncorrelated alpha engines — an **IPO Penny Stock Momentum Sleeve** and a **Multi-Period Convex Portfolio Optimization Sleeve** — into a single unified capital allocation framework.

The system was designed to maximize risk-adjusted returns on an unseen out-of-sample dataset (2020–2025), after being calibrated and validated on a 10-year in-sample development period (2010–2020). All constraints mirror realistic market conditions: fixed initial capital of ₹50,00,000, a 100-stock portfolio cap, no fractional shares, and a 26.8 bps one-way transaction cost (53.6 bps round-trip).

---

## Table of Contents

- [Performance Results](#performance-results)
- [Architecture Overview](#architecture-overview)
- [Sleeve 1: IPO Penny Stock Momentum](#sleeve-1-ipo-penny-stock-momentum)
- [Sleeve 2: Multi-Period Convex Portfolio Optimization](#sleeve-2-multi-period-convex-portfolio-optimization)
- [Stock Selector — Feature Engineering](#stock-selector--feature-engineering)
- [Capital Allocation & Integration Logic](#capital-allocation--integration-logic)
- [Implementation Details](#implementation-details)
- [Dependencies & Installation](#dependencies--installation)
- [Usage](#usage)
- [Output Files](#output-files)
- [Mathematical Appendix](#mathematical-appendix)

---

## Performance Results

Evaluated over a **10-year in-sample backtest** (December 31, 2009 → January 1, 2020):

| Metric | Strategy | Nifty 500 |
|---|---|---|
| **CAGR** | **37.43%** | 8.61% |
| Annualized Volatility | 17.39% | 15.05% |
| Maximum Drawdown | **-25.08%** | -31.82% |
| **Sharpe Ratio** | **1.947** | 0.633 |
| **Sortino Ratio** | **2.301** | — |
| Information Ratio vs Nifty 500 | **1.274** | — |
| Up-Capture | 52.9% | — |
| Down-Capture | **19.6%** | — |
| Annualized Turnover | 14.3× | — |

### Rolling Outperformance vs Nifty 500

| Window | Average Outperformance | Worst Underperformance |
|---|---|---|
| 1-Year | 33.3% | -0.7% |
| 3-Year | 190.4% | +77.7% |
| 5-Year | 582.5% | +307.2% |

> The strategy never underperformed the Nifty 500 over any rolling 3-year or 5-year window in the backtest period.

### Key Highlights

- **4.35× benchmark CAGR** (37.43% vs 8.61%) with lower maximum drawdown (-25.08% vs -31.82%)
- **Asymmetric capture profile**: 52.9% upside capture with only 19.6% downside capture, yielding a **2.69× Capture Ratio**
- **Sharpe of 1.947** — nearly 3× the benchmark's 0.633
- Consistent alpha across all rolling horizons despite a 14.3× annualized turnover rate, demonstrating that the hysteresis filter successfully absorbs transaction friction

---

## Architecture Overview

```
Initial Capital (₹50,00,000)
        │
        ├─── IPO Momentum Sleeve (30%)
        │       ├── Universe filtering (price < ₹40, age = 5 days)
        │       ├── Hierarchical priority queue (fresh → missed → TSL-hit)
        │       ├── Trailing stop-loss (20% from HWM)
        │       └── Drawdown-triggered capital transfer → Core Sleeve
        │
        └─── Convex Optimization Sleeve (70%)
                ├── Feature selection (6-factor composite score)
                ├── Hysteresis-based turnover control (enter=65, exit=79)
                ├── cvxportfolio MultiPeriodOptimization (horizon=2)
                ├── Worst-case risk penalization
                └── T+1 integer-share execution
```

Capital weights adapt dynamically: when the stock universe exceeds 1,200 equities on inception day, the split shifts defensively to **10% IPO / 90% Core** to reduce idiosyncratic noise exposure.

---

## Sleeve 1: IPO Penny Stock Momentum

### Core Thesis

Newly listed Indian micro-cap stocks exhibit short-lived, asymmetric momentum breakouts after the initial listing-day price bands expire. By systematically capturing this window and employing hard risk gates, this sleeve targets high idiosyncratic alpha.

### Universe Construction

**Five-Day Stability Rule**: A stock enters the candidate set only after surviving the initial five-day listing window — ensuring institutional flipping has subsided and a genuine market equilibrium is established:

```
Cₜ = { i ∈ Uₜ | age_i(t) = 5 AND i ∉ NSE500 }
```

**Price Ceiling Filter**: A hard upper price limit of ₹40 per share isolates penny stocks with high price elasticity, where marginal capital movement generates disproportionate percentage returns:

```
Φ(i, t) = 𝟙{ P_{i,t} ≤ 40 }
```

### Capacity Scaling

The number of simultaneous position slots scales with backtest duration to ensure breadth without over-dilution:

```
N_slots(Y) = clip(⌊5 + Y⌉, 5, 20)
```

where Y is the backtest duration in years. This allows the strategy to expand breadth across multiple IPO cycles in longer datasets.

### Position Sizing & Capital Utilization

Each new entry receives a proportional allocation based on remaining slot capacity:

```
w_init = K_IPO,t / (N_slots - N_occupied)
```

A **single-position cap** of σ = 10% of total IPO portfolio value is enforced. When `w_init > σ · V_IPO,t`, the excess capital is recursively redistributed — **90% of the overflow is reinvested proportionally across existing holdings** — minimizing cash drag.

### Risk Management

**Asset-Level Trailing Stop-Loss (TSL)**: Maintains a per-position High Water Mark and triggers liquidation at a 20% retrace:

```
S_exit,t = 𝟙{ P_{i,t} ≤ 0.80 · M_{i,t} }
```

**Sleeve-Level Drawdown Intervention** (two-tier):

| Threshold | Action |
|---|---|
| 30% drawdown from HWM | Liquidate 50% of all IPO positions; transfer proceeds to Core Sleeve |
| 40% drawdown from HWM | Full exit of all IPO positions; halt new entries; transfer all cash to Core Sleeve |

This cross-sleeve capital routing ensures losses in the high-risk IPO engine cannot compromise the overall fund's viability.

### Hierarchical Priority Queue

When a position slot becomes available, the engine sweeps three candidate pools in strict priority order:

1. **Incoming Queue** (`Q_incoming`) — Fresh listings at t = t₀(i) + 5; highest priority for capturing initial momentum breakout
2. **Missed Opportunity Queue** (`Q_missed`) — Stocks that met criteria but were blocked by full capacity; processed LIFO (most recent first, as they retain more remaining potential)
3. **TSL-Hit Recovery Queue** (`Q_hit`) — Stocks exited via trailing stop; re-entry requires passing a dual hysteresis filter:
   - **Temporal cooldown**: ≥5 trading days since exit
   - **Breakout confirmation**: Current price > Exit price × 1.10 (filters dead-cat bounces)
   - **Trading cap**: Max 10 re-entries per ticker (prevents over-trading stagnant assets)

---

## Sleeve 2: Multi-Period Convex Portfolio Optimization

### Approach

The core portfolio is driven by `cvxportfolio`, implementing a **Multi-Period Optimization (MPO)** policy with a planning horizon of 2 periods. Rather than static mean-variance optimization, the engine simultaneously solves for current and next-period trades, allowing forward-looking cost avoidance.

### Objective Function

The optimizer maximizes a composite risk-penalized objective:

```
maximize:  ReturnsForecast
         - 0.05 × ReturnsForecastError
         - 15 × (WorstCaseRisk([DiagonalCovariance, FactorModelCovariance])
                 + 0.01 × RiskForecastError)
```

Key design choices:

- **Worst-Case Risk**: The optimizer selects the maximum risk across two covariance models (diagonal and factor-model), preventing over-reliance on any single specification — a form of distributional robustness.
- **Return Forecast Error Penalty**: Discourages over-concentration in assets with uncertain alpha signals.
- **Risk Forecast Error Penalty**: Discourages concentrated exposure in assets with highly unstable covariance estimates.

### Constraints

Applied at every rebalance date:

- `LeverageLimit(1)` — gross exposure ≤ 1 (no borrowing)
- `LongOnly(applies_to_cash=True)` — no short positions, no negative cash
- `MaxWeights(universe_mask)` — only hysteresis-filtered universe assets may hold positive weight

### Multi-Period Extension (Horizon = 2)

**Intertemporal Coupling**: The two-period objective couples trades across periods through shared inventory:

```
w_{t+1} = w_t + Δw_t   (inventory transition)
```

This means the engine anticipates next-period transaction costs when deciding today's trades — reducing unnecessary churn.

**Receding-Horizon Execution**:
1. At time t, forecast returns and risk
2. Solve the 2-period optimization
3. Execute only the first period's trades
4. Re-solve at t+1 with updated market data

**Trading Frequency**: Weekly rebalancing reduces the number of optimization calls, stabilizes covariance estimation, and naturally limits turnover.

---

## Stock Selector — Feature Engineering

### Six-Factor Composite Score

For each asset i at time t, the selector computes six normalized signals:

| Factor | Formula | Weight |
|---|---|---|
| Long-Term Momentum | (P_{i,t} / P_{i,t-60}) - 1 | +0.30 |
| Medium-Term Momentum | (P_{i,t} / P_{i,t-20}) - 1 | +0.20 |
| Annualized Volatility | σ₆₀ × √252 | -0.30 |
| Volume Liquidity | 20-day SMA of traded volume | +0.10 |
| NATR (14-day) | (ATR₁₄ / Price) × 100 | +0.05 |
| Rolling Beta | Cov(Rᵢ, Rₘ) / Var(Rₘ) over 60d | +0.05 |

Each factor is **cross-sectionally Z-scored** and clipped to [-3, 3] daily:

```
Z_{f,i,t} = clip( (f_{i,t} - μ_{f,t}) / σ_{f,t}, -3, 3 )
```

The composite score is:

```
S_{i,t} = 0.30·Z_Mom60 + 0.20·Z_Mom20 - 0.30·Z_σ + 0.10·Z_Liq + 0.05·Z_NATR + 0.05·Z_β
```

### Hysteresis-Based Turnover Control

A naive daily top-N ranking would cause excessive boundary churn. The hysteresis filter introduces a buffer zone:

- **Entry**: Rank ≤ 65 → stock enters eligible universe
- **Hold**: Rank < 79 → stock remains in universe
- **Exit**: Rank ≥ 79 → stock is removed

The exit threshold of 79 is intentionally calibrated so that the Core Sleeve (max 79 positions) combined with the IPO Sleeve never exceeds the competition's 100-stock hard limit.

---

## Capital Allocation & Integration Logic

### Default Split

```python
IPO_ALLOC  = 0.30   # 30% → IPO Momentum Sleeve
PF_ALLOC   = 0.70   # 70% → Core Convex Optimization Sleeve
```

### Large Universe Adjustment

```python
if day1_stock_count > 1200:
    IPO_ALLOC, PF_ALLOC = 0.10, 0.90
```

### Cross-Sleeve Capital Transfer

When the IPO Sleeve triggers its drawdown-based liquidation protocol, proceeds are injected into the Core Sleeve as a **cash injection** at the next available rebalance date. The backtester scales forward allocations proportionally to absorb the additional capital without disrupting the optimization structure.

---

## Implementation Details

### Data Handling

The pipeline auto-detects two possible column schema formats (development vs competition) and normalizes them:

```python
# Development schema:  tradedate, fid, traded_value, traded_volume, gics_sector, ...
# Competition schema:  date, symbol, value, volume, sector, lms, in_nse500, ...
```

Both schemas are accepted and internally unified. Execution price is computed as the OHLC average:

```python
exec_price = (open + high + low + close) / 4.0
```

### T+1 Execution

All signals are computed on day t; trades are executed at day t+1 prices. The backtester enforces this via a strict execution mapping:

```python
execution_map[price_index[signal_loc + 1]] = signal_date
```

### Integer Share Constraint

Continuous weight allocations are converted to floor-integer share quantities:

```
q_t = ⌊ w_t* × V_t / p̄_{t+1} ⌋
```

A minimum 1-share position is maintained at all times to satisfy the competition's minimum position constraint.

### Transaction Cost Model

```
Φ(Δqₜ) = τ × ‖Δqₜ ⊙ p̄_{t+1}‖₁,   τ = 0.00268 (26.8 bps one-way)
```

The cash recursion prevents negative balances:

```
X_{t+1} = max(X_t - Δq_t^T × p̄_{t+1} - Φ(Δqₜ), 0)
```

### No-Leverage Enforcement

Before execution, allocations are post-processed to scale down stock weights proportionally if the implied cash balance turns negative — ensuring the portfolio never uses leverage.

---

## Dependencies & Installation

```bash
pip install cvxpy cvxportfolio pandas numpy matplotlib
```

Optional (for live benchmark download):

```bash
pip install yfinance
```

### Required Data File

The strategy expects an NSE price file in one of the following formats:

```
nse_prices_complete 1.parquet   (primary)
nse_prices_complete.parquet     (fallback)
nse_prices_complete.csv         (fallback)
```

Required columns (competition schema): `date`, `symbol`, `open`, `high`, `low`, `close`, `volume`, `in_nse500`

---

## Usage

```bash
python 2613_CodeBase_QuantFinChallenge.py
```

The script runs all 7 stages sequentially and prints progress at each step:

```
[1/7] Loading data...
[2/7] Running IPO strategy...
[3/7] Running portfolio strategy...
[4/7] Combining sleeves...
[5/7] Benchmark...
[6/7] Metrics...
[7/7] Outputs...
```

### Key Configuration Constants

```python
INITIAL_CAPITAL      = 5_000_000     # ₹50 lakhs
TXN_COST_PCT         = 0.00268       # 26.8 bps one-way
MAX_TOTAL_POSITIONS  = 100           # hard portfolio cap

# IPO Sleeve
IPO_MAX_PRICE        = 40.0          # ₹ price ceiling for entries
IPO_TRAILING_STOP    = 0.20          # 20% TSL from HWM
IPO_DD_HALF          = 0.3           # 30% DD → 50% liquidation
IPO_DD_FULL          = 0.4           # 40% DD → full exit

# Stock Selector
ENTER_N = 65                         # entry rank threshold
EXIT_N  = 79                         # exit rank threshold
```

---

## Output Files

All outputs are written to `strategy_output/`:

| File | Description |
|---|---|
| `performance_report.txt` | Full metric table printed to file |
| `equity_curve.png` | Strategy vs Nifty 500 NAV on log scale |
| `drawdown_curve.png` | Drawdown profile comparison |
| `position_count.png` | Daily position count over time |
| `turnover.png` | Daily turnover as % of NAV |
| `rolling_outperformance.png` | Rolling 1-year alpha vs benchmark |
| `monthly_heatmap.png` | Calendar-view monthly returns heatmap |
| `trade_log.csv` | Full IPO sleeve trade-by-trade log |
| `daily_nav.csv` | Daily NAV, positions, benchmark, drawdowns |
| `rolling_outperformance.csv` | Rolling window summary table |
| `metrics_summary.csv` | All KPIs in machine-readable format |

---

## Mathematical Appendix

### Composite Score Weights (Rationale)

| Factor | Weight | Rationale |
|---|---|---|
| Long Momentum (60d) | +0.30 | Primary trend signal; highest predictive power |
| Short Momentum (20d) | +0.20 | Medium-term confirmation of trend |
| Volatility (60d) | -0.30 | Penalizes erratic assets; equal magnitude to long momentum |
| Liquidity (20d) | +0.10 | Scalability filter; minimizes slippage |
| NATR (14d) | +0.05 | Secondary volatility proxy at shorter horizon |
| Beta (60d) | +0.05 | Systematic risk exposure |

### IPO Slot Capacity Formula

```
N_slots = clip(⌊5 + Y⌉, 5, 20)
```

Example: a 3-year dataset → 8 slots; a 10-year dataset → 15 slots.

### Drawdown Calculation

```
DD_t = (HWM_t - V_t) / HWM_t
HWM_t = max_{τ ≤ t} V_τ
```

### Trailing Stop-Loss Per Position

```
M_{i,t} = max_{τ ∈ [t_entry, t]} P_{i,τ}
exit_signal_t = 𝟙{ P_{i,t} ≤ (1 - λ) · M_{i,t} },   λ = 0.20
```

---

## Project Structure

```
MultiPeriodConvex-main/
├── 2613_CodeBase_QuantFinChallenge.py        # Full strategy implementation
├── 2613_DocmenReport_QuantFinChallenge.pdf   # Technical documentation (18 pages)
└── 2613_PerformanceReport_QuantFinChallenge.pdf  # Results & diagnostics
```

---

## Strategy Design Philosophy

The central insight driving this architecture is that a **high-friction environment (53.6 bps round-trip) demands asymmetric alpha capture rather than raw return maximization**. The system is explicitly engineered around three principles:

1. **Volatility minimization over return maximization**: The convex objective strictly penalizes variance, forecast error, and covariance instability. The system targets risk-parity-aligned allocations, not momentum-chasing ones.

2. **Turnover control at every layer**: The hysteresis-based universe filter, weekly rebalancing frequency, and multi-period optimization horizon all independently reduce unnecessary trades. This is the primary lever for preserving net-of-cost alpha.

3. **Asymmetric drawdown protection**: The IPO sleeve's two-tier liquidation protocol and per-asset trailing stop-losses work in concert with the convex sleeve's worst-case risk penalty to create a portfolio that participates less in upside but substantially caps downside — producing the 52.9% / 19.6% capture profile and a 2.69× capture ratio.

The result is a strategy that compounds at 37.43% CAGR while keeping maximum drawdown below 25% — a risk/return profile that consistently and significantly outperforms the Nifty 500 across all rolling measurement windows.

---

*KRITI 2026 Quantitative Finance Challenge · Hostel 2613*
