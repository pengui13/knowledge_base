# Volatility — Core Concepts

## Definition

**Volatility** refers to the statistical measure of **dispersion of returns** or values over time.

It is measured as the **standard deviation of log returns**:

$$\sigma = \sqrt{\frac{1}{N-1} \sum_{i=1}^{N} (r_i - \bar{r})^2}$$

where $r_i = \ln\left(\dfrac{P_i}{P_{i-1}}\right)$ are the **log returns** of a price series.

---

## Why Log Returns?

They are used because they are:

- **Time-additive** — you can sum them across periods
- **Approximately normally distributed** — which makes math cleaner
- **Symmetric** — a 50% gain and 50% loss don't cancel out with simple returns, but do with log returns

---

## The Natural Logarithm — Intuition via Compounding

Consider $\$1000$ invested at an annual rate of **100%**. How much do you have after 1 year?

It depends on how frequently interest is compounded:

| Compounding | Formula | Result |
|---|---|---|
| 1× per year | $1000 \cdot (1 + 1)^1$ | $\$2000$ |
| 2× per year | $1000 \cdot \left(1 + \frac{1}{2}\right)^2$ | $\$2250$ |
| 4× per year | $1000 \cdot \left(1 + \frac{1}{4}\right)^4$ | $\approx \$2441$ |
| $n \to \infty$ | $1000 \cdot \lim_{n \to \infty}\left(1 + \frac{1}{n}\right)^n$ | $1000 \cdot e \approx \$2718$ |

The limit defines Euler's number:

$$e = \lim_{n \to \infty}\left(1 + \frac{1}{n}\right)^n \approx 2.718$$

### Interpreting $e$ and $\ln$

> If $e$ answers *"by what factor does something grow in one unit of time under continuous compounding?"*,  
> then $\ln$ answers *"how long did it take for something to grow by that factor?"*

- $1000 \cdot e^1$ — value after **1 year**
- $1000 \cdot e^2$ — value after **2 years**
- $e$ is essentially a **multiplier of continuous growth**

**How much continuous growth happened when price went from 100 to 200?**

$$r = \ln\!\left(\frac{200}{100}\right) = \ln(2) \approx 0.693$$

This means: under continuous compounding, **69.3% growth** over the period — that is your log return.

---

## Annualizing Volatility

Daily volatility is typically scaled to an annual figure for comparison across assets and time periods.

$$\sigma_{\text{annual}} = \sigma_{\text{daily}} \times \sqrt{252}$$

where **252** is the conventional number of trading days in a year.

> This scaling follows from the **square-root-of-time rule**: if daily returns are independent and identically distributed, variance grows linearly with time, so standard deviation grows with $\sqrt{T}$.

$$\text{Var}_{T} = T \cdot \text{Var}_{1} \implies \sigma_T = \sigma_1 \cdot \sqrt{T}$$

# Chapter 2. Theoretical Foundations and Data Preparation

## 2.1 Stationarity of Time Series

### 2.1.1 Definition

A time series $\{X_t\}$ is said to be **weakly (covariance) stationary** if the following three conditions hold simultaneously:

$$\mathbb{E}[X_t] = \mu \qquad \text{(constant mean)}$$

$$\text{Var}(X_t) = \sigma^2 \qquad \text{(constant variance)}$$

$$\text{Cov}(X_t,\, X_{t+k}) = \gamma(k) \qquad \text{(autocovariance depends only on lag } k \text{, not on } t \text{)}$$

Weak stationarity requires that the statistical structure of the series does not shift over time. Strong stationarity imposes a stricter condition — that the entire joint distribution is time-invariant — but in practice, weak stationarity is sufficient for most econometric models.

---

### 2.1.2 Expectation

The **expected value** (expectation) of a random variable $X$ is the probability-weighted average of all possible outcomes — the value one would observe on average over an infinite number of trials:

$$\mathbb{E}[X] = \sum_{x} x \cdot P(X = x) \qquad \text{(discrete case)}$$

$$\mathbb{E}[X] = \int_{-\infty}^{\infty} x \cdot f(x)\, dx \qquad \text{(continuous case)}$$

**Example.** For a fair six-sided die, no single roll produces the expected value, yet over many trials the average converges to it:

$$\mathbb{E}[X] = 1 \cdot \frac{1}{6} + 2 \cdot \frac{1}{6} + 3 \cdot \frac{1}{6} + 4 \cdot \frac{1}{6} + 5 \cdot \frac{1}{6} + 6 \cdot \frac{1}{6} = 3.5$$

Applied to the stationarity condition $\mathbb{E}[X_t] = \mu$: regardless of the point in time $t$, the average value of the series must remain the same. As an illustration, the S&P 500 index price violates this condition — its mean grew from approximately 350 in 1990 to 5000 in 2024 — whereas the daily log returns fluctuate around a mean of approximately zero across all observed periods:

$$\mathbb{E}[P_t] \neq \mathbb{E}[P_{t+k}] \quad \Rightarrow \quad \text{price series: NOT stationary}$$

$$\mathbb{E}[r_t] \approx 0 \quad \forall\, t \quad \Rightarrow \quad \text{log returns: stationary in mean} \checkmark$$

---

### 2.1.3 Variance

**Variance** quantifies the degree of dispersion of a random variable around its mean:

$$\text{Var}(X) = \mathbb{E}\!\left[(X - \mu)^2\right]$$

Intuitively, each observation is subtracted from the mean, squared (to eliminate sign), and averaged. The result measures how widely the values are spread.

The stationarity condition $\text{Var}(X_t) = \sigma^2$ requires this spread to be constant over time. In the context of financial returns, this property is frequently violated: daily S&P 500 swings reached $\pm 3$–$5\%$ during the 2008 financial crisis, while remaining near $\pm 0.3\%$ during the calm period of 2017.

This phenomenon — time-varying conditional variance — is termed **heteroskedasticity**:

$$\text{Var}(r_t \mid r_{t-1}, r_{t-2}, \ldots) = \sigma_t^2 \neq \text{const}$$

It is important to distinguish between **unconditional** and **conditional** variance. The unconditional variance of log returns remains finite and stable over long horizons, satisfying the stationarity requirement. The conditional variance, however, changes over time and constitutes the primary object of study in volatility modelling. This distinction is precisely what GARCH-family and regime-switching models are designed to exploit.

---

### 2.1.4 Autocovariance

The third stationarity condition concerns the **autocovariance** — the covariance of the series with a lagged version of itself:

$$\text{Cov}(r_t,\, r_{t+k}) = \gamma(k)$$

Here, $r_t$ denotes the return at time $t$ and $r_{t+k}$ the return $k$ periods later. The condition states that this relationship depends **only on the lag** $k$, not on the absolute position in time $t$. Concretely:

$$\text{Cov}(r_1, r_4) = \text{Cov}(r_{100}, r_{103}) = \text{Cov}(r_{1000}, r_{1003}) = \gamma(3)$$

The economic interpretation of $\gamma(k)$ is as follows:

| Sign of $\gamma(k)$ | Interpretation |
|---|---|
| $\gamma(k) > 0$ | Returns exhibit **momentum** — a move in one direction tends to persist after $k$ periods |
| $\gamma(k) < 0$ | Returns exhibit **mean reversion** — a move tends to reverse after $k$ periods |
| $\gamma(k) = 0$ | Returns carry **no memory** — past returns have no predictive power at lag $k$ |

All three cases are consistent with stationarity. What would break stationarity is instability of $\gamma(k)$ across time — for instance, if returns show strong positive autocorrelation during crises (panic selling reinforces itself) but zero autocorrelation during calm periods. In that case, the autocovariance structure depends on $t$, violating the condition.

This is precisely the tension addressed by regime-switching models: within a single regime, $\gamma(k)$ remains stable; across regimes it may differ. A pooled model that ignores regime membership will exhibit apparent non-stationarity — providing the core motivation for the regime-based framework developed in this thesis.

---

### 2.1.5 Why Non-Stationary Series Cannot Be Modelled Reliably

A model trained on non-stationary data fits the trend rather than the underlying data-generating process, and therefore lacks predictive validity. Three specific failure modes arise:

**Spurious regression.** Two unrelated non-stationary series that share an upward trend will appear strongly correlated. A model will identify a relationship that has no causal basis and will not generalise.

**Meaningless parameter estimates.** The mean and variance estimated from a trending series describe no particular moment in time. They represent an average over a constantly shifting distribution.

**Model assumption violation.** GARCH, Hidden Markov Models, and regression-based approaches all require a stable data-generating process. If the mean is drifting, the model misinterprets systematic trend as volatility shocks, producing distorted estimates.

The transformation to log returns resolves the mean non-stationarity. The residual heteroskedasticity — time-varying conditional variance — is not a problem to be eliminated but a signal to be modelled.

---

## 2.2 Data Acquisition and Preparation

### 2.2.1 Dataset

The empirical analysis is conducted on daily closing prices of the S&P 500 index (ticker: `^GSPC`) from January 1990 to the present, sourced via the `yfinance` library. The dataset encompasses approximately 9,000 trading days, covering multiple distinct market cycles including the Dot-com crash (2000–2002), the Global Financial Crisis (2008–2009), the COVID-19 shock (2020), and subsequent recovery periods.

### 2.2.2 Implementation

The following libraries are used throughout the analysis:

```python
import pandas as pd
import numpy as np
import yfinance as yf
import matplotlib.pyplot as plt
from statsmodels.tsa.stattools import adfuller
from sklearn.cluster import KMeans
```

**Data download and cleaning:**

```python
sp500 = yf.download(
    "^GSPC",
    start="1990-01-01",
    auto_adjust=True,
    progress=False
)

sp500 = sp500.sort_index()
sp500 = sp500[["Close"]].dropna()
```

The resulting dataset contains adjusted closing prices indexed by date. The first five observations are shown below:

| Date | Close |
|---|---|
| 1990-01-02 | 359.69 |
| 1990-01-03 | 358.76 |
| 1990-01-04 | 355.67 |
| 1990-01-05 | 352.20 |
| 1990-01-08 | 353.79 |

**Log return computation:**

Daily log returns are computed as the first difference of the log-transformed price series:

$$r_t = \ln P_t - \ln P_{t-1} = \ln\!\left(\frac{P_t}{P_{t-1}}\right)$$

```python
sp500["log_return"] = np.log(sp500["Close"]).diff()
sp500 = sp500.dropna()
```

The resulting series for the first five observations:

| Date | Close | log\_return |
|---|---|---|
| 1990-01-03 | 358.76 | −0.002589 |
| 1990-01-04 | 355.67 | −0.008650 |
| 1990-01-05 | 352.20 | −0.009804 |
| 1990-01-08 | 353.79 | +0.004504 |
| 1990-01-09 | 349.62 | −0.011857 |

**Descriptive statistics of the log return series:**

```python
sp500["log_return"].describe()
```

| Statistic | Value |
|---|---|
| Count | 9,067 |
| Mean | 0.000325 |
| Std | 0.011384 |
| Min | −0.127652 |
| 25th percentile | −0.004436 |
| Median | 0.000602 |
| 75th percentile | 0.005694 |
| Max | 0.109572 |

The mean log return is near zero (0.03% daily), consistent with the stationarity in mean requirement. The standard deviation of approximately 1.14% constitutes the baseline volatility estimate. The minimum of −12.77% corresponds to the largest single-day crash in the sample period.

---

### 2.2.3 Visual Comparison: Price vs. Log Returns

The distinction between a non-stationary and a stationary series is most clearly illustrated visually. Figure 2.1 presents both series over the full sample period.

```python
fig, ax = plt.subplots(2, 1, figsize=(10, 6), sharex=True)

ax[0].plot(sp500.index, sp500["Close"])
ax[0].set_title("S&P 500 Price")
ax[0].grid(True)

ax[1].plot(sp500.index, sp500["log_return"])
ax[1].set_title("S&P 500 Log Returns")
ax[1].grid(True)

plt.tight_layout()
plt.show()
```

![alt text](image.png)

The upper panel exhibits a clear upward trend with expanding variance — characteristic of a non-stationary process. The lower panel fluctuates around zero with no discernible trend, though periods of elevated variance (notably 2008–2009 and 2020) are clearly visible. This **volatility clustering** — the tendency for large moves to follow large moves — is the empirical phenomenon that motivates GARCH and regime-switching modelling.

---

### 2.2.4 Stationarity Test: Augmented Dickey-Fuller

To formally verify the stationarity of the log return series, the **Augmented Dickey-Fuller (ADF) test** is applied. The null hypothesis of the test is the presence of a unit root (non-stationarity). Rejection at the 5% significance level ($p < 0.05$) provides statistical evidence of stationarity.

```python
p_value = adfuller(sp500["log_return"].dropna())[1]
```

A $p$-value substantially below 0.05 confirms that the log return series is stationary in mean, satisfying the prerequisite for the volatility models applied in subsequent chapters.

# Chapter 3. Empirical Analysis of S&P 500 Volatility Regimes

## 3.1 Volatility Clustering

### 3.1.1 Motivation

A fundamental empirical property of financial return series is **volatility clustering** — the tendency for large price movements to be followed by large movements, and calm periods to persist. This phenomenon, first documented by Mandelbrot (1963) and later formalized by Engle (1982), implies that the conditional variance of returns is not constant over time but exhibits serial dependence.

To provide an initial visual inspection of this property, the absolute value of daily log returns is computed:

$$|r_t| = \left|\ln\frac{P_t}{P_{t-1}}\right|$$

Absolute returns serve as a model-free proxy for daily volatility. Periods of elevated $|r_t|$ indicate turbulent market conditions, while prolonged sequences of small $|r_t|$ indicate calm regimes.

```python
sp500["abs_return"] = sp500["log_return"].abs()

sp500["abs_return"].plot(
    title="Absolute Log-Returns (Volatility Clustering)",
    figsize=(10, 4)
)
```

The resulting chart reveals pronounced clustering of large absolute returns around two historically significant events: the **Global Financial Crisis of 2008–2009** and the **COVID-19 market shock of March 2020**. Outside these periods, absolute returns remain consistently small, indicating structurally different market behavior. This visual evidence motivates the formal regime-detection framework developed in subsequent sections.

---

## 3.2 Rolling Realized Volatility

### 3.2.1 Definition and Construction

While absolute returns provide a day-by-day measure of realized movement, a smoother estimate of the local volatility level is obtained through **rolling realized volatility** — the standard deviation of log returns computed over a fixed trailing window, scaled to an annualized figure.

For a window of $T = 21$ trading days (approximately one calendar month), the rolling volatility at time $t$ is defined as:

$$\hat{\sigma}_t = \sqrt{252} \cdot \sqrt{\frac{1}{T-1} \sum_{i=0}^{T-1} \left(r_{t-i} - \bar{r}_{t}\right)^2}$$

where $\bar{r}_t$ denotes the mean log return within the window, and the factor $\sqrt{252}$ annualizes the estimate under the assumption of 252 trading days per year, following the square-root-of-time rule.

```python
sp500["vol_21d"] = (
    sp500["log_return"]
    .rolling(21)
    .std()
    * np.sqrt(252)
)

sp500["vol_21d"].plot(
    title="Rolling Volatility (21 days, annualized)",
    figsize=(10, 4)
)
```

### 3.2.2 Choice of Window Length

The 21-day window represents a deliberate methodological balance. Shorter windows (e.g., 5–10 days) react rapidly to market shocks but introduce substantial noise, producing erratic regime assignments. Longer windows (e.g., 63 days — one quarter) yield smoother estimates but lag significantly behind actual regime transitions, failing to detect crises in a timely manner. The 21-day window is standard in the empirical finance literature and is adopted here as the primary volatility estimator.


![alt text](telegram-cloud-photo-size-2-5249062983839717965-y.jpg)

### 3.2.3 Observed Patterns

The rolling volatility series exhibits several empirically important features consistent with the heteroskedasticity hypothesis:

The annualized volatility fluctuates between approximately **5–15%** during calm market conditions and exceeds **60–80%** during crisis episodes. The 2008 financial crisis and the March 2020 COVID shock represent the two most extreme volatility regimes in the sample, with the latter briefly reaching values above **90% annualized**. Extended calm periods — notably 2004–2006 and 2017 — show volatility persistently below **15%**, confirming the structural separation between market states.

This evidence directly supports the central hypothesis of the present thesis: that a single unconditional volatility estimate is an inadequate characterization of S&P 500 risk, and that a regime-based framework captures the underlying dynamics more accurately.

---


## 3.3 Regime Detection via K-Means Clustering

### 3.3.1 Methodology

To formally partition the volatility series into distinct regimes, the **K-Means clustering algorithm** is applied to the rolling volatility estimates $\hat{\sigma}_t$. K-Means identifies $k$ cluster centroids $\{\mu_1, \mu_2, \mu_3\}$ by minimizing the within-cluster sum of squares:

$$\underset{C}{\arg\min} \sum_{j=1}^{k} \sum_{\hat{\sigma}_t \in C_j} \left(\hat{\sigma}_t - \mu_j\right)^2$$

Three clusters are specified ($k = 3$), corresponding to the three qualitatively distinct market states observable in Figure 3.2: low volatility, medium volatility, and high volatility (crisis).

### 3.3.2 Regime Labeling

Following the fitting procedure, clusters are ordered by their centroid values and assigned interpretable labels:

```python
regime_map = {
    order[0]: "Low Volatility",
    order[1]: "Medium Volatility",
    order[2]: "High Volatility"
}

sp500_reg = sp500.dropna(subset=[('regime', '')]).copy()
sp500_reg[('regime_label', '')] = sp500_reg[('regime', '')].map(regime_map)
```

The resulting regime assignments align closely with known historical periods of market stress, providing face validity for the clustering approach. The **Low Volatility** regime dominates the sample, covering the majority of trading days. The **High Volatility** regime captures crisis episodes with high precision, correctly identifying 2008–2009 and 2020 as structurally distinct periods.
![alt text](telegram-cloud-photo-size-2-5249062983839717995-y.jpg)
---

## 3.4 Value-at-Risk Analysis

### 3.4.1 Naive VaR

**Value-at-Risk (VaR)** at confidence level $1 - \alpha$ is defined as the loss threshold exceeded with probability $\alpha$:

$$\text{VaR}_\alpha = Q_\alpha(r_t)$$

where $Q_\alpha$ denotes the $\alpha$-quantile of the return distribution. At the conventional $\alpha = 0.05$ level, VaR answers the question: *on the worst 5% of trading days, what is the minimum loss incurred?*

The **naive VaR** is estimated unconditionally — treating all 9,067 observations as draws from a single stationary distribution:

```python
alpha = 0.05
VaR_naive = sp500["log_return"].quantile(alpha)
```

This produces a single threshold applied uniformly across all market conditions, regardless of the prevailing volatility regime.

### 3.4.2 Regime-Conditional VaR

The naive approach is theoretically inconsistent with the heteroskedasticity documented in Section 3.2. If the conditional distribution of returns differs across regimes, a single unconditional threshold will be simultaneously too conservative during calm periods and insufficiently cautious during crises.

The **regime-conditional VaR** addresses this by estimating a separate quantile within each regime:

$$\text{VaR}_\alpha^{(s)} = Q_\alpha\left(r_t \mid s_t = s\right), \quad s \in \{0, 1, 2\}$$

```python
VaR_regime = (
    sp500["log_return"]
    .groupby(sp500["regime"])
    .quantile(alpha)
)
```

Each trading day is then assigned the VaR threshold corresponding to its detected regime:

```python
sp500["VaR_regime_day"] = sp500["regime"].map(VaR_regime)
```

### 3.4.3 Breach Analysis

A **VaR breach** occurs when the realized return falls below the VaR threshold — an event that should occur with probability exactly $\alpha = 5\%$ under a correctly specified model:

```python
sp500["breach_naive"]  = sp500["log_return"] < VaR_naive
sp500["breach_regime"] = sp500["log_return"] < sp500["VaR_regime_day"]
```

The aggregate breach counts are as follows:

| Model | Total Breaches | Expected (5%) |
|---|---|---|
| Naive VaR | 455 | ~453 |
| Regime VaR | 453 | ~453 |

The near-identical total breach counts reflect a property of quantile estimation by construction — both models target the 5th percentile of their respective distributions, so aggregate coverage is similar. The critical distinction lies not in the total count but in the **temporal distribution** of breaches across market regimes.

### 3.4.4 Regime-Conditional Breach Rates

The economically meaningful comparison is the breach rate **within each regime**. A well-calibrated model should produce a breach rate close to 5% in every regime. A miscalibrated model will exhibit systematic over- or under-coverage in specific market states.

```python
sp500.groupby("regime")[["breach_naive", "breach_regime"]].mean() * 100
```

The naive VaR, calibrated on the full unconditional distribution, applies a threshold derived primarily from calm-period observations to crisis-period days. Consequently, it **severely underestimates tail risk during crises** — producing a breach rate well below 5% in the high-volatility regime — while **overestimating risk during calm periods**, where the threshold is unnecessarily strict. The regime-conditional VaR corrects this systematic bias by applying crisis-appropriate thresholds during crisis regimes and relaxed thresholds during calm periods.

### 3.4.5 Temporal Distribution of Breaches

The annual distribution of breaches provides the most compelling visual evidence of model misspecification. The naive VaR produces extreme breach concentrations in 2008 (~49 breaches) and 2020 (~30 breaches) — approximately four times the expected annual rate of ~12 breaches. These spikes reflect the fundamental failure of a static threshold to adapt to structural changes in market volatility.

The regime-conditional VaR distributes breaches more evenly across years, demonstrating that regime-awareness substantially reduces the model's vulnerability to being blindsided during market crises. This result constitutes the primary empirical contribution of the present analysis: **regime-based risk estimation provides more consistent tail coverage across varying market conditions than conventional unconditional VaR**.