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
