# Detailed Methodology and Results

[Back to the main README](../README.md)

Gaussian vs Bootstrap Monte Carlo models, with corporate-action-adjusted NSE data and an out-of-sample VaR backtest.

**Stocks:** RELIANCE, TCS, ICICIBANK, BHARTIARTL, LT  |  **Data:** NSE daily closes, 01-Jan-2021 to 31-Dec-2025  |  **Tools:** Python, pandas, NumPy, SciPy, matplotlib

---

## TL;DR

- Built a Monte Carlo risk engine for an equal-weight ₹10 lakh portfolio of 5 large-cap Indian stocks using two models: a **Gaussian (parametric)** model and a **bootstrap (non-parametric)** model, plus a **block-bootstrap** variant.
- Raw NSE prices contain corporate-action jumps (a fake **-49.8%** "crash" from Reliance's 1:1 bonus). I identified every price-changing event from the official NSE corporate actions list, adjusted for them, and validated the result against Yahoo Finance (daily-return correlation 0.995 or higher for all 5 stocks).
- Real returns are **fat-tailed**: the portfolio had 17 days beyond 3 standard deviations, against about 3.3 expected under a normal curve. This is why a Gaussian model alone is not enough.
- The Gaussian model gave a **1-day 99% CVaR of 2.47%**, while the bootstrap gave **3.63%** (about 47% higher). The models agree on typical outcomes and differ in the extreme tail.
- Models were fitted on 2021-2024 only and **backtested on unseen 2025 data** with the Kupiec test. No model was rejected, but the Gaussian model **understated the size of its worst breaches**. A second split (test on 2024-2025, which includes the 4-Jun-2024 election crash) confirmed this.

---

## 1. The question

> If I invest ₹10 lakh equally in these 5 stocks, what could it be worth in one year, and how bad can a loss get?

Nobody can predict the future, so Monte Carlo simulation creates thousands of possible futures (10,000 here, each 252 trading days long) from how the stocks behaved in the past, and then studies the whole range of outcomes, including the bad ones.

---

## 2. Data

| Item | Detail |
|---|---|
| Source | NSE historical price data (one CSV per stock) |
| Period | 01-Jan-2021 to 31-Dec-2025 |
| Series | EQ (normal equity trading) only |
| Rows | 1,240 trading days per stock (1,239 daily returns) |
| Price used | `Close Price` (the NSE `Prev Close` column is **not** adjusted for corporate actions, so it is not used) |
| Corporate actions | Official NSE Corporate Actions CSV for each stock |
| Cross-check | Yahoo Finance (`auto_adjust=True`) |

The NSE data contains **5 weekend trading sessions** (Diwali Muhurat on 12-Nov-2023, and special Saturday sessions on 20-Jan-2024, 02-Mar-2024, 18-May-2024 and 01-Feb-2025). These are real trading days and are kept.

---

## 3. Data cleaning and corporate actions

A model should see changes in **value**, not changes in share count. If raw prices are used, a 1:1 bonus looks like a 50% crash and ruins every volatility and risk number.

I did not rely on memory or a jump detector. The workflow was:

1. Check data quality (dates, duplicates, gaps, missing values, stale prices).
2. Load the official NSE corporate actions list and filter for price-changing events (bonus, split, rights, demerger).
3. Look at the returns on those dates **before** adjusting.
4. Apply adjustment factors to all prices **before** each ex-date.
5. Look at the same dates **after** adjusting.

**Price-changing events found in 2021-2025 (the official list contained exactly these three):**

| Stock | Event | Ex-date | Factor | Return on ex-date before → after |
|---|---|---|---|---|
| RELIANCE | Demerger (Jio Financial Services) | 20-Jul-2023 | 2580 / 2841.85 ≈ 0.9079 | -7.81% → +1.54% |
| RELIANCE | Bonus 1:1 | 28-Oct-2024 | 0.5 | -49.76% → +0.49% |
| BHARTIARTL | Rights issue 1:14 at ₹535 | 27-Sep-2021 | TERP / cum-rights close ≈ 0.98157 | -2.28% → -0.44% |

- The demerger factor uses the ₹2,580 price discovered for Reliance in the NSE special pre-open session on 20-Jul-2023, against the previous close of ₹2,841.85.
- The Airtel rights issue is small and would **not** be caught by a simple "big jump" scan (its z-score was only -1.6). It is only found through the official corporate actions list.
- TCS and L&T buybacks do not change the price mechanically. ICICI Bank had no price-changing events.
- **Dividends are not adjusted.** All analysis is on a price-return basis (see Limitations).

---

## 4. Validation against Yahoo Finance

Adjusted NSE log returns were compared with Yahoo Finance's adjusted returns on 1,235 common days.

| Stock | Correlation of daily returns |
|---|---|
| RELIANCE | 0.9991 |
| TCS | 0.9947 |
| ICICIBANK | 0.9950 |
| BHARTIARTL | 0.9995 |
| LT | 0.9956 |

Every day where the two sources differed by more than 0.5% was investigated. All differences have an explanation:

| Cause | Explanation |
|---|---|
| **Dividends** | Yahoo's adjusted series adds dividends back; mine is price-only. The gap is negative on dividend ex-dates, matched against the NSE dividend list. This also explains why TCS's cumulative line sits lower in the comparison chart. |
| **Weekend sessions** | NSE has 5 weekend sessions that Yahoo does not. The next trading day's return then differs by minus the weekend return. |
| **Yahoo date shift** | On 18/19-Mar-2025, TCS, ICICIBANK, LT and BHARTIARTL show offsetting gaps (+x then -x), meaning one source has the move one day late. NSE is the primary source, so the NSE values were kept. |
| **Reliance demerger** | Yahoo did adjust, but with a slightly different factor (a 1.65% gap on 20-Jul-2023). The NSE-based factor was kept. |

---

## 5. Exploratory findings

| | Annualised volatility | Excess kurtosis | Days beyond 3σ |
|---|---|---|---|
| RELIANCE | 22.7% | 2.78 | 13 |
| TCS | 20.8% | 2.44 | 16 |
| ICICIBANK | 21.9% | 8.08 | 14 |
| BHARTIARTL | 22.6% | 1.70 | 15 |
| LT | 24.0% | 6.92 | 12 |
| **Equal-weight portfolio** | **15.3%** | **4.55** | **17** |

A normal distribution would produce about **3.3** days beyond 3σ out of 1,239.

1. **Fat tails.** All series have positive excess kurtosis and the Jarque-Bera test rejects normality (p ≈ 0). QQ-plots bend away from the line at both ends.
2. **Volatility is not constant.** The portfolio's 21-day rolling annualised volatility ranges from about 6.4% to 31.9%, and squared returns are autocorrelated (lag 1: 0.185, against a noise band of ±0.062).
3. **Diversification.** Pairwise correlations are 0.22-0.43, so the portfolio volatility (15.3%) is far lower than any single stock (21-24%).
4. **Mean reversion in this sample.** Variance ratios are below 1 (0.82 at 21 days, 0.66 at 63 days), which matters for the block bootstrap (see Results).

---

## 6. Methodology

### 6.1 Train/test split

| Set | Period | Daily returns |
|---|---|---|
| Train | 2021-2024 | 990 |
| Test | 2025 | 249 |

All model parameters come from the **train** set only. The test set is kept hidden until the backtest.

### 6.2 Model 1: Gaussian (parametric)

Assumes daily log returns are multivariate normal with mean vector μ and covariance matrix Σ estimated from the training data. Correlated random shocks are produced with the **Cholesky decomposition** of Σ:

```python
L = np.linalg.cholesky(cov)
Z = np.random.standard_normal((n_sims, n_days, 5))
daily = mu + Z @ L.T
```

### 6.3 Model 2: Bootstrap (non-parametric)

No distribution assumption. Whole trading days (all 5 stocks together) are resampled with replacement from the training data, so the real correlations and the real fat tails are preserved automatically.

### 6.4 Model 2b: Block bootstrap

Instead of single days, chunks of consecutive days are resampled so that short-term patterns such as volatility clustering are kept. Block lengths of 5, 10 and 20 days were tested; **10 days** is the main version.

### 6.5 Portfolio and risk metrics

- Starting value ₹10,00,000, split equally (20% each), buy-and-hold over the simulated year. 10,000 paths × 252 days. Fixed random seed (42).
- **VaR** (Value at Risk) at 95% and 99%, and **CVaR** (Expected Shortfall: the average loss in the worst cases beyond VaR). Reported as positive loss fractions.
- Probability of loss and the distribution of maximum drawdown.

### 6.6 Backtest

- The 1-day VaR for each model is estimated from all simulated days (about 2.5 million), using the training data only.
- Compared with the **actual** 2025 equal-weight daily portfolio return. A **breach** is a day where the loss exceeds VaR.
- At 95% about 5% of days should breach, and at 99% about 1%.
- The **Kupiec proportion-of-failures test** checks whether the number of breaches is statistically consistent with the expected rate (p-value above 0.05 means the model is not rejected).
- Because Kupiec only counts breaches, the **average loss on breach days** is also compared with the model's CVaR, to check the *size* of the tail losses.
- A **robustness split** (train 2021-2023, test 2024-2025) gives 498 test days and includes the 4-Jun-2024 election-result crash (-6.8%).

### 6.7 Why there are different time periods

| Number | What it is |
|---|---|
| 252 days | Length of each simulated future (one trading year) |
| 1-year VaR/CVaR | Risk over the whole simulated year (investor's view) |
| 1-day VaR/CVaR | Risk over a single day. This is what can be backtested, because 2025 has 249 daily observations but only one "year". |
| 95% / 99% | Confidence levels, not time periods |
| Block length 5/10/20 | Size of the chunk of consecutive days copied in the block bootstrap (about 1, 2 and 4 weeks). A model setting, not a horizon. |
| 21-day rolling window | Used only to chart how volatility changes over time |

---

## 7. Results

### 7.1 Simulation results (trained on 2021-2024, ₹10 lakh, 10,000 paths, 252 days)

| Metric | Gaussian | Bootstrap | Block 5d | Block 10d | Block 20d |
|---|---|---|---|---|---|
| Mean 1-year return | 24.2% | 24.4% | 24.2% | 23.8% | 24.0% |
| Std of 1-year return | 19.9% | 19.8% | 18.8% | 18.5% | 17.8% |
| 1-year VaR 95% | 5.4% | 5.2% | 4.4% | 4.5% | 3.2% |
| 1-year CVaR 95% | 10.9% | 11.4% | 10.1% | 10.3% | 8.9% |
| 1-year VaR 99% | 14.1% | 15.5% | 13.5% | 14.0% | 12.2% |
| 1-year CVaR 99% | 18.0% | 19.8% | 18.0% | 18.4% | 16.8% |
| 1-day VaR 95% | 1.51% | 1.45% | 1.40% | 1.41% | 1.46% |
| 1-day CVaR 95% | 1.92% | 2.16% | 2.05% | 2.13% | 2.23% |
| 1-day VaR 99% | 2.17% | 2.48% | 2.48% | 2.53% | 2.82% |
| 1-day CVaR 99% | **2.47%** | **3.63%** | 3.44% | 3.55% | 3.77% |

Reading the table:

- **Gaussian vs bootstrap:** the 95% 1-day numbers are almost the same, but at the extreme end the Gaussian CVaR 99% is about a third lower. The Gaussian is fine for an ordinary bad day and too optimistic for a very bad one.
- **Block bootstrap shows lower 1-year risk.** The training period shows partial mean reversion (variance ratio 0.82 at 21 days), which blocks preserve and independent draws destroy. Because this grows with block length and the sample is only about 4 years, it is likely partly specific to this sample, and should not be read as the portfolio being safer.
- The mean 1-year return of about 24% reflects the 2021-2024 bull run (annualised means: Airtel 29%, L&T 26%, ICICI 23%, TCS 8.5%, Reliance 7.6%). It is **not a forecast**. The risk metrics are the focus of this project.
- All models share the same training drift, so differences between them come from the shape of the distribution and not from different return assumptions.

### 7.2 Backtest results

**Main split: train 2021-2024, test 2025 (249 days)**

| Model | Level | VaR | CVaR | Expected breaches | Actual breaches | Kupiec p-value | Avg loss on breach |
|---|---|---|---|---|---|---|---|
| Gaussian | 95% | 1.54% | 1.95% | 12.5 | 7 | 0.085 | 2.13% |
| Gaussian | 99% | 2.20% | 2.53% | 2.5 | 2 | 0.747 | 2.87% |
| Bootstrap | 95% | 1.41% | 2.12% | 12.5 | 8 | 0.167 | 2.06% |
| Bootstrap | 99% | 2.53% | 3.59% | 2.5 | 1 | 0.281 | 3.34% |
| Block 10d | 95% | 1.41% | 2.12% | 12.5 | 8 | 0.167 | 2.06% |
| Block 10d | 99% | 2.53% | 3.59% | 2.5 | 1 | 0.281 | 3.34% |

**Robustness split: train 2021-2023, test 2024-2025 (498 days)**

| Model | Level | VaR | CVaR | Expected breaches | Actual breaches | Kupiec p-value | Avg loss on breach |
|---|---|---|---|---|---|---|---|
| Gaussian | 95% | 1.53% | 1.93% | 24.9 | 16 | 0.051 | 2.36% |
| Gaussian | 99% | 2.19% | 2.52% | 5.0 | 6 | 0.656 | 3.34% |
| Bootstrap | 95% | 1.45% | 2.07% | 24.9 | 19 | 0.206 | 2.22% |
| Bootstrap | 99% | 2.53% | 3.30% | 5.0 | 3 | 0.336 | 4.32% |
| Block 10d | 95% | 1.47% | 2.09% | 24.9 | 18 | 0.136 | 2.26% |
| Block 10d | 99% | 2.53% | 3.30% | 5.0 | 3 | 0.336 | 4.32% |

---

## 8. Conclusions

1. **No model was rejected** by the Kupiec test (all p-values above 0.05). Every model had *fewer* breaches than expected, because 2025 was calmer than the training years (daily portfolio volatility 0.82% against 0.99%). The models were conservative, not dangerous. The Gaussian 95% result on the second split (p = 0.051) is borderline because it had too **few** breaches.
2. **The Gaussian model matched the frequency of bad days but understated their size.** On its 99% breach days, the average loss was 2.87% against the model's CVaR of 2.53% (main split), and 3.34% against 2.52% (second split, about 33% worse than predicted).
3. **The bootstrap captures tail severity better but is capped by its history.** A bootstrap cannot produce a day worse than the worst day in its training data. On 04-Jun-2024 the portfolio lost 6.8%, which exceeded the bootstrap's average tail loss (4.32% average breach against a 3.30% CVaR).
4. **Bootstrap and block bootstrap give almost the same 1-day numbers**, since blocks only change how days are linked over several days. A 1-day backtest cannot separate them.
5. **Corporate-action handling matters.** Unadjusted data would have created a fake -49.8% day and distorted every risk figure.

---

## 9. Limitations

- **Price returns only.** Dividends are not included (typically 0.3-2% a year per stock; TCS's large special dividends are visible in the Yahoo comparison).
- **Short, bullish training window.** Four years of data, during which the mean return was about 24% a year. The mean is not a forecast. A conservative-drift variant was left as a sensitivity check.
- **Low statistical power.** The 2025 test has 249 days and the 99% level has only 1-2 breaches, so the backtest cannot rank the models with confidence.
- **Static VaR.** Parameters are estimated once. A real risk desk re-estimates on a rolling window.
- **Independent-day assumption.** The Gaussian and simple bootstrap models ignore volatility clustering. No GARCH-type model was fitted.
- **Simple portfolio.** Equal weights, buy-and-hold, no transaction costs, no rebalancing, no taxes or slippage.
- **Manual adjustments.** The demerger factor depends on the special-session price, and Yahoo's own factor differs slightly.

---

## 10. Reproducing the results

```
monte-carlo-portfolio-risk/
├── README.md
├── requirements.txt
├── notebooks/
│   └── monte_carlo_portfolio_risk.ipynb     # full analysis, run top to bottom
├── data/
│   ├── raw/                                 # NSE price CSVs (RELIANCE, TCS, ICICIBANK, BHARTIARTL, L_T)
│   ├── corporate_actions/                   # NSE corporate actions CSVs (CA_*.csv)
│   └── processed/                           # adjusted_prices.csv, log_returns.csv
├── images/                                  # charts used in the README
└── docs/
    └── METHODOLOGY.md                       # this file
```

```bash
pip install pandas numpy scipy matplotlib yfinance
```

Settings: 10,000 simulations, 252 trading days, ₹10,00,000, equal weights, random seed 42. Notebook sections: data loading and cleaning → corporate actions → returns, validation and exploration → train/test split → Gaussian model → bootstrap models → backtest.

---

## 11. Glossary (plain English)

| Term | Meaning |
|---|---|
| **Monte Carlo simulation** | Running a random experiment many times and studying the spread of results |
| **Log return** | `ln(today's price / yesterday's price)`, a way of measuring daily change that adds up over time |
| **Corporate action** | A company event (bonus, split, rights issue, demerger) that changes the share price without changing value |
| **Adjustment factor** | A number that rescales old prices so they are comparable with current ones |
| **Covariance / correlation** | How strongly the stocks move together |
| **Cholesky decomposition** | A matrix technique that turns independent random numbers into correlated ones |
| **Fat tails / kurtosis** | Extreme moves happen more often than a normal curve predicts |
| **Bootstrap** | Re-drawing real historical days at random to build new possible futures |
| **VaR** | The loss that should not be exceeded on 95% (or 99%) of occasions |
| **CVaR / Expected Shortfall** | The average loss in the worst cases beyond VaR |
| **Backtest** | Checking a model's predictions against data it has never seen |
| **Breach** | A day where the actual loss was worse than the predicted VaR |
| **Kupiec test** | A statistical test of whether the number of breaches is consistent with the expected rate |

---

## References and disclaimer

- NSE India: historical price data and corporate actions.
- Yahoo Finance (via `yfinance`): cross-validation of adjusted returns.
- Kupiec, P. (1995). *Techniques for Verifying the Accuracy of Risk Measurement Models.* Journal of Derivatives.

This is an educational project. It is not investment advice, and simulated results are not predictions of future performance.
