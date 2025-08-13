# Pairs Spread Z-Score Indicator (Pairs Z+)

## Overview
The **Pairs Spread Z-Score** indicator is a statistical trading tool designed for **pairs trading** — a market-neutral strategy that exploits the relative mean-reverting behavior between two correlated assets.  
It calculates the spread between two assets, standardizes it into a Z-score, and highlights extreme deviations from the mean that may present **long** or **short** reversion opportunities.

The script supports **fixed or rolling beta calculation**, optional **log price transformation**, and a **close-only mode** to ensure backtesting and bot safety.  
It also calculates the **half-life of mean reversion**, providing insight into how quickly spreads tend to revert.

---

##  Methodology

1. **Pair Selection**  
   The trader selects two tickers (Y = dependent, X = independent) that historically move together (high correlation, cointegration preferred).

2. **Spread Calculation**  
   Formula:  
   `Spread = Price(Y) − β × Price(X)`  
   where β is either:
   - Fixed (user-defined)
   - Auto OLS (calculated via rolling ordinary least squares regression)

3. **Log Transformation (Optional)**  
   Prices can be transformed into natural logs to normalize percentage-based relationships.

4. **Z-Score Computation**  
   Formula:  
   `Z = (Spread − Mean(Spread)) / StdDev(Spread)`  
   This measures how far the current spread deviates from its historical mean in standard deviation units.

5. **Signal Generation**  
   - **Long Signal** → Z <= Lower Entry Threshold for N consecutive bars  
   - **Short Signal** → Z >= Upper Entry Threshold for N consecutive bars  
   - **Exit Signal** → |Z| <= Exit Threshold for N consecutive bars

6. **Half-Life Estimation**  
   Uses AR(1) regression to estimate the number of bars for half the spread deviation to decay, giving insight into the speed of mean reversion.

---

## Tunable Inputs and Configurations

- **Ticker 1 (Y)** – The dependent asset whose relative mispricing is being measured. The Z-score is based on *this* asset’s spread vs. Ticker 2.  
  *Example:* In a JPM vs. XLF trade, JPM is Y.

- **Ticker 2 (X)** – The independent asset used as the reference. Its price movements form the baseline for calculating the spread.  
  *Example:* In a JPM vs. XLF trade, XLF is X.

- **Beta Mode** – Determines how β (hedge ratio) is calculated.  
  - *Fixed*: Uses a static β (good for stable long-term relationships).  
  - *Auto OLS*: Dynamically recalculates β using rolling regression (adapts to changing relationships).

- **Beta (Fixed Mode)** – The β multiplier when Fixed mode is chosen.  
  *Example:* β = 1.2 means you short $12,000 of X for every $10,000 long in Y.

- **Beta Lookback (Auto OLS)** – Number of bars used to estimate β in Auto OLS mode.  
  *Tip:* Longer lookbacks = more stable β, shorter lookbacks = more reactive.

- **Use Log Prices?** – Toggles whether prices are transformed into their natural logarithm before calculation.  
  *Why:* Log prices normalize percentage moves, which is important when assets differ in price scale.

- **Z-Score Lookback** – Number of bars used to calculate the rolling mean and standard deviation of the spread.  
  *Impact:* Smaller values = more sensitive (noisier), larger values = smoother but slower signals.

- **Upper Entry Threshold** – Positive Z-score level above which a short Y / long X trade is signaled.  
  *Example:* 2.0 = Y is “too expensive” relative to X.

- **Lower Entry Threshold** – Negative Z-score level below which a long Y / short X trade is signaled.  
  *Example:* −2.0 = Y is “too cheap” relative to X.

- **Exit Threshold** – Absolute Z-score level where open trades are closed.  
  *Example:* 0.5 means closing trades when spread is near the mean.

- **Confirm Bars** – Number of consecutive bars that must meet the entry/exit condition before a signal is triggered.  
  *Why:* Filters out false spikes.

- **Half-Life Lookback** – Bars used for estimating the mean-reversion half-life.  
  *Tip:* Helps gauge if the spread is reverting quickly enough for your trade horizon.

- **Close-Only Mode** – If enabled, calculations only use completed candles.  
  *Why:* Eliminates intrabar repainting, making it safe for bots/backtests.

---

## Features

- Close-only evaluation to prevent repainting and ensure historical accuracy  
- Auto-alignment of time series from different symbols  
- Null-safe calculations for live and historical data  
- Real-time and historical signal labeling  
- Diagnostics labels showing current β, Z-score, and half-life

---

##  How to Trade This Indicator

### 1. Core Concept
In this setup, **Ticker Y** is the dependent variable, meaning we measure and trade based on its relative overpricing or underpricing versus **Ticker X**.  
If the spread is **positive and high**, Y is expensive relative to X.  
If the spread is **negative and low**, Y is cheap relative to X.

### 2. Basic Long/Short Logic
- **Go Long Y / Short X**  
  When Z-score <= Lower Entry Threshold (e.g., -2), Y is considered *cheap* vs. X and historically likely to revert upward.  
  The trade profits if Y outperforms X in the future.

- **Go Short Y / Long X**  
  When Z-score >= Upper Entry Threshold (e.g., +2), Y is considered *expensive* vs. X and historically likely to revert downward.  
  The trade profits if Y underperforms X in the future.

### 3. Exit Logic
Exit both legs when the spread mean-reverts to near zero (|Z| <= Exit Threshold).

### 4. Position Sizing
To remain market neutral:  
`Dollar value in Y ≈ β × Dollar value in X`  
Example: If β = 1.2 and you short $12,000 of X, you’d long $10,000 of Y.

### 5. Practical Checklist
1. Confirm Y/X is **cointegrated** (using the Python scanner below).  
2. Choose sensible thresholds (±2 for entry, ±0.5–1 for exit).  
3. Ensure half-life is not excessively long — otherwise, trades may stagnate.  
4. Avoid thinly traded assets; liquidity is critical for execution.  
5. Keep leverage modest until the method is proven.

---

## ⚠️ Caveats & Best Practices

- **Close-Only Mode** is strongly recommended for automated strategies  
- Best results are achieved with pairs that are **cointegrated**, not just correlated  
- Thresholds should be tuned to the volatility of the spread — too low and signals will be noisy, too high and trades may be rare  
- Half-life is an estimate — use it as a guide, not an absolute rule

---

## FAQ

**Q: Does this indicator repaint?**  
A: In Close-Only Mode, no. All calculations are based on completed bars.

**Q: Why use Z-scores instead of raw spreads?**  
A: Z-scores standardize the spread, making threshold signals consistent across different scales.

**Q: How is this different from Bollinger Band spreads?**  
A: Functionally similar in detecting extremes, but Z-score standardization is more precise and integrates easily with statistical modeling.

**Q: What is β (beta) in this context?**  
A: β adjusts the scale of X relative to Y to better model their relationship; it’s the slope from regressing Y on X.

---

## Example Usage

1. Choose two highly correlated tickers (e.g., XLF & JPM)  
2. Set Beta Mode to Auto OLS with a 120-bar lookback  
3. Use Z-Score lookback of 60 bars, Entry thresholds at ±2, Exit at ±0.5  
4. Enable Close-Only Mode for bot safety  
5. Trade long when Z <= −2 and short when Z >= 2, exiting when |Z| <= 0.5

---

## Reference: Python Cointegrated Pairs Scanner

This indicator pairs with the **[Python Cointegrated Pairs Scanner](https://github.com/Gustivus/cointegrated-pairs-scanner)** (`cointegration_scanner_2.1.py`).  
That script:
- Scans an index (e.g., S&P 500) for highly correlated & cointegrated pairs
- Runs Engle–Granger tests with FDR correction
- Outputs trade-ready pairs with hedge ratios, spread volatility, and ADF stationarity checks

**Workflow**:
1. Run the Python scanner to generate a shortlist of statistically viable pairs.  
2. Load those tickers into this TrendSpider indicator to actively monitor Z-score signals.  
3. Execute trades only when scanner-validated pairs trigger live signals.

By using both together, you ensure **data-driven pair selection** (Python) + **timely trade execution** (TrendSpider).

---

##  License
MIT License – free to use, modify, and distribute. 
