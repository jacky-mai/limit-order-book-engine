# Limit Order Book Reconstruction & Optimal Execution Engine

**Difficulty:** Institutional
**Categories:** Market Microstructure · Algorithmic Trading

A from-first-principles implementation of a price-time-priority limit order
book (LOB) matching engine, microstructure analytics, and two canonical
institutional execution models — Almgren-Chriss optimal execution and
Avellaneda-Stoikov market making — built and validated in a single Colab
notebook.

---

## 1. Project Overview

This project reconstructs a limit order book from a tick-level event stream,
computes the microstructure metrics that market-making and execution desks
monitor in real time, and implements two of the most cited quantitative
finance results in market microstructure theory:

- **Almgren-Chriss (2000/2001):** the optimal trade-scheduling framework
  that balances market-impact cost against price-path risk when liquidating
  a large position.
- **Avellaneda-Stoikov (2008):** the inventory-aware market-making model
  that derives the optimal bid/ask quotes around a risk-adjusted
  reservation price.

Because free-tier data providers do not expose genuine Level-2 depth data,
the notebook ships with a calibrated **synthetic order-flow simulator** as
its primary data source, built on a real price-time priority matching
engine (not a toy approximation) — including correct handling of
marketable/crossing limit orders, an easy source of correctness bugs in
naive LOB implementations. An optional loader for real **LOBSTER** academic
sample data is included so the same downstream pipeline runs unchanged on
real tick data.

## 2. Real-World Finance Use Case

Managing execution cost and inventory risk under uncertain, adversarial
order flow is the central problem for market-making and execution desks:

- **Market makers** (Citadel Securities, Jane Street, Optiver, Virtu
  Financial, Hudson River Trading) continuously quote two-sided prices and
  must manage inventory risk against adverse selection — the exact problem
  Avellaneda-Stoikov formalizes.
- **Execution desks** (agency brokers, buy-side trading desks at asset
  managers, and bank program-trading desks) must liquidate or acquire large
  positions while minimizing market impact — the exact problem
  Almgren-Chriss formalizes, and the basis for most institutional
  implementation-shortfall and VWAP/TWAP scheduling algorithms in use today.
- **Microstructure metrics** (spread, order-book imbalance, Kyle's lambda,
  noise-corrected realized volatility) are the standard real-time
  diagnostics that both of the above desks monitor to calibrate their
  models intraday.

## 3. System Architecture

```
                         ┌────────────────────────┐
                         │   DATA LAYER            │
                         │  - Synthetic simulator  │
                         │  - LOBSTER loader (opt.)│
                         └───────────┬─────────────┘
                                     │ event stream
                                     ▼
                         ┌────────────────────────┐
                         │  MATCHING ENGINE        │
                         │  LimitOrderBook class   │
                         │  (price-time priority,  │
                         │   crossing-order logic) │
                         └───────────┬─────────────┘
                                     │ snapshots + trades
                                     ▼
              ┌──────────────────────┴───────────────────────┐
              ▼                                              ▼
   ┌─────────────────────────┐                  ┌─────────────────────────┐
   │  MICROSTRUCTURE LAYER   │                  │   STRATEGY LAYER        │
   │  - Spread / imbalance   │                  │  - Almgren-Chriss       │
   │  - Kyle's lambda        │                  │    optimal execution    │
   │  - Realized vol / TSRV  │                  │  - Avellaneda-Stoikov   │
   └───────────┬─────────────┘                  │    market making        │
               │                                └────────────┬────────────┘
               └───────────────────┬───────────────────────── ┘
                                   ▼
                     ┌───────────────────────────┐
                     │  VISUALIZATION & REPORTING│
                     │  matplotlib charts +       │
                     │  consolidated summary table│
                     └───────────────────────────┘
```

## 4. Required APIs and Data Sources

| Source | Role | Notes |
|---|---|---|
| Synthetic simulator (built-in) | Primary data source | Always available, no API key required; calibrated Poisson order-flow model |
| [LOBSTER](https://lobsterdata.com) | Optional real-data source | Free academic sample data (NASDAQ, limited tickers/days); requires manual download |
| Yahoo Finance / Polygon.io | Not required for this notebook | Referenced only as context for where genuine tick data would come from in production; free tiers do not provide adequate L2 depth history |

## 5. Required Python Libraries

- `numpy`, `pandas` — numerical computing and data handling
- `scipy` — OLS regression (Kyle's lambda), numerical routines
- `matplotlib` — all charts
- `nbformat` — only needed if rebuilding the notebook programmatically, not required to run it

See `requirements.txt`.

## 6. Folder/File Structure

Even though the deliverable runs as a single Colab notebook, it is organized
as if it were a GitHub repository so sections can be lifted into standalone
modules later:

```
lob-execution-engine/
├── README.md
├── requirements.txt
├── LOB_Reconstruction_Optimal_Execution.ipynb   # main deliverable
├── src/                        # (conceptual — all classes live in the
│   ├── engine.py                #  notebook's Section 2A/2B cells; split
│   ├── microstructure.py        #  out here if productionizing)
│   ├── execution.py
│   └── market_making.py
├── data/
│   └── lobster_samples/        # place downloaded LOBSTER CSVs here
└── outputs/
    └── figures/                 # exported chart images, if saved from Colab
```

## 7. Step-by-Step Build Guide

1. Open the notebook in Google Colab; run Section 1 (imports) and Section 2
   (configuration).
2. Run Section 2A to define the `LimitOrderBook` matching engine and
   `LOBSimulator` order-flow generator.
3. (Optional) Point Section 2B at real LOBSTER files and set
   `USE_REAL_LOBSTER_DATA = True` to swap in real tick data.
4. Run Section 3 to execute the simulation and reconstruct book snapshots.
5. Run Section 4 to compute microstructure metrics (spread, imbalance,
   Kyle's lambda, realized vol vs. TSRV).
6. Run Section 5 to generate book-dynamics and microstructure visualizations.
7. Run Section 6 to compute the Almgren-Chriss optimal execution trajectory,
   Monte Carlo cost simulation, and efficient frontier.
8. Run Section 7 to simulate the Avellaneda-Stoikov market maker against a
   naive fixed-spread benchmark, including a risk-aversion sensitivity sweep.
9. Run Section 8 for the consolidated performance summary table.
10. Adjust `CONFIG` at the top of the notebook to explore different
    liquidity, volatility, or risk-aversion regimes without touching any
    downstream code.

## 8. Data Collection Pipeline

- **Synthetic path:** independent Poisson processes drive limit-order
  arrivals, market-order arrivals, and cancellations. New limit orders are
  priced relative to the **live current mid-price** of the book (not a
  fixed anchor), so price evolves endogenously from order-flow imbalance.
  The book is bootstrapped with initial resting liquidity so it is never
  empty at t=0.
- **Real-data path:** `load_lobster_events()` parses a LOBSTER message file
  (`Time, Type, OrderID, Size, Price, Direction`) into the same internal
  event schema used by the synthetic path, including the documented
  price-scale conversion (raw integer price / 10,000). Missing files raise
  a clear, actionable error rather than failing silently.

## 9. Data Cleaning & Feature Engineering

- Trade prices are resampled onto a regular time grid via last-observed-price
  forward fill (the standard "previous tick" scheme) before realized-variance
  estimation, since raw tick timestamps are irregular.
- Order book snapshots are only recorded once both a best bid and best ask
  exist, avoiding undefined mid-price/spread values during bootstrap.
- Per-level depth is captured (not just aggregate depth) to support the
  depth-heatmap visualization and enable future feature extraction (e.g.,
  order flow toxicity metrics, VPIN).
- A defensive invariant check flags any crossed-book state
  (`best_bid > best_ask`), which should be structurally unreachable given
  the crossing-order matching logic — a check every reviewer of a LOB
  engine should expect to see.

## 10. Core Models/Algorithms

| Model | Purpose | Key formula |
|---|---|---|
| Price-time priority matching engine | Book reconstruction | FIFO per price level; crossing limit orders match immediately |
| Kyle's (1985) lambda | Price impact per unit signed volume | `Δmid = λ · signed_volume + ε` (rolling OLS) |
| Two-Scale Realized Volatility (Zhang, Mykland & Aït-Sahalia, 2005) | Noise-corrected volatility estimation | `TSRV = RV_sparse_avg − (n̄/n)·RV_all_ticks` |
| Almgren-Chriss (2000/2001) | Optimal execution trajectory | `x_j = X·sinh(κ(T−t_j)) / sinh(κT)` |
| Avellaneda-Stoikov (2008) | Inventory-aware market making | `r = s − qγσ²(T−t)`, `δ = γσ²(T−t) + (2/γ)ln(1+γ/κ)` |

All four non-trivial formulas are validated against known theoretical
limits in the notebook itself (e.g., the Almgren-Chriss trajectory exactly
recovers TWAP as λ→0; inventory variance provably falls as Avellaneda-Stoikov
risk aversion rises).

## 11. Visualizations & Dashboard Components

- Reconstructed mid-price and bid-ask spread over time
- Order book imbalance time series
- Bid/ask depth-by-level heatmap
- Naive vs. two-scale (noise-corrected) realized variance comparison
- Rolling Kyle's lambda
- Almgren-Chriss trajectory vs. TWAP
- Almgren-Chriss efficient frontier (cost vs. risk)
- Execution cost distribution histogram (TWAP vs. AC-optimal)
- Avellaneda-Stoikov quotes, inventory, and P&L vs. naive benchmark
- Inventory/P&L volatility sensitivity to risk aversion
- Consolidated performance summary table (dashboard-style)

## 12. Performance Metrics

- **Microstructure:** mean spread, mean |imbalance|, mean Kyle's lambda,
  naive-RV/TSRV inflation ratio
- **Execution:** expected cost and cost standard deviation for TWAP vs.
  AC-optimal trajectories, generated via Monte Carlo simulation under the
  Almgren-Chriss price-impact dynamics
- **Market making:** final P&L, P&L standard deviation, and maximum
  absolute inventory for the Avellaneda-Stoikov strategy vs. a naive
  fixed-spread benchmark, across a grid of risk-aversion levels

## 13. Final Deliverables

- `LOB_Reconstruction_Optimal_Execution.ipynb` — fully executed Colab
  notebook (all 26 cells run with zero errors), production-quality comments,
  input validation, and defensive invariant checks throughout
- `README.md` — this document
- `requirements.txt` — pinned minimum dependency versions

## 14. Resume Description

*Built a limit order book reconstruction and optimal execution research
platform in Python: implemented a price-time-priority matching engine,
noise-corrected (two-scale) realized volatility and Kyle's lambda price-impact
estimation, and the Almgren-Chriss optimal execution and Avellaneda-Stoikov
market-making models, validated against known theoretical limits and
benchmarked against naive TWAP/fixed-spread strategies via Monte Carlo
simulation.*

## 15. Potential Upgrades

- Replace the synthetic simulator with real LOBSTER or vendor tick data end-to-end
- Extend Almgren-Chriss to multi-asset execution with cross-impact terms
- Add adverse-selection-aware market making (Guéant-Lehalle-Fernandez-Tapia model)
- Wrap the pipeline in a Streamlit/Plotly Dash app for a live interactive dashboard
- Backtest Avellaneda-Stoikov quotes directly against the reconstructed book
  from Section 3 instead of a freshly simulated price path
- Add order-flow toxicity metrics (e.g., VPIN) using the per-level depth data already captured
