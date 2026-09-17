# Claims Frequency & Severity Monte Carlo Forecast

Monte Carlo simulation for forecasting insurance claims **frequency** (claims per
policy, %) and **severity** (average cost per claim, $) over a 5-year horizon,
using monthly historical data (Jan-2020–Dec-2026).

The model combines a classical log-linear trend + seasonal decomposition with an
AR(1) residual process, correlates frequency and severity shocks, and propagates
both **residual uncertainty** and **parameter (estimation) uncertainty** through
10,000 simulated paths via a sieve bootstrap.

## What this does

1. **Decomposition** — fits `log(y_t) = trend + seasonal(month) + residual` by OLS
   for both series.
2. **Residual dynamics** — fits an AR(1) model to each series' residuals, plus the
   contemporaneous correlation between frequency and severity shocks (they tend
   to move together — inflation, claims-mix shifts, catastrophe years affect
   both).
3. **Sieve bootstrap** — resamples the AR(1) innovations, reconstructs pseudo-
   history, and re-fits the whole model 500 times to get a distribution over the
   trend/seasonal/AR/correlation parameters, not just point estimates.
4. **Monte Carlo simulation** — 10,000 simulated 60-month paths, each drawing its
   own bootstrapped parameter set plus correlated, autocorrelated random shocks.
5. **Diagnostics** — Q-Q plots and Shapiro-Wilk tests on the AR(1) innovations to
   check the Gaussian-shock assumption the whole simulation rests on.
6. **Outputs** — percentile fan charts, annual roll-ups, empirical distributions
   and CDFs at 1/3/5-year horizons, and raw simulation arrays for downstream use
   (e.g. pure premium calculations).

## Results preview

5-year (2031) annual average forecast, median with 90% interval (P5–P95):

| | Median | P5–P95 interval |
|---|---|---|
| **Frequency** | 16.5% | 14.0% – 18.9% |
| **Severity** | $4,950 | $4,470 – $5,550 |

Accounting for parameter uncertainty (via the sieve bootstrap) rather than
residual noise alone widens the severity P5–P95 band by **+41% at the 5-year
horizon** (Dec-2031: $953 → $1,346), versus only +8% at the 1-year horizon —
uncertainty in the fitted trend compounds the further out you forecast.

**Forecast fan chart** — historical series plus simulated median and P5–P95 /
P25–P75 bands, fixed-parameter vs. bootstrap-augmented:

![Fan chart](fan_chart_bootstrap_comparison.png)

**Diagnostic check** — Q-Q plots of the AR(1) innovations against a Normal
distribution. Both series show a heavy left-tail outlier (the Jan-2026
severity/frequency jump), which is why Shapiro-Wilk rejects normality at the
5% level for both — a reason to treat simulated tail percentiles (P5/P95) as
indicative rather than precise:

![Q-Q plot](qq_innovations.png)

Full monthly and annual percentile tables are in
`simulation_summary_bootstrap.csv` and `annual_summary_bootstrap.csv`.

## Repo contents

| File | Description |
|---|---|
| `mc_bootstrap.py` | Main script: decomposition, bootstrap, simulation, diagnostics, plots |
| `simulation_summary_bootstrap.csv` | Monthly forecast percentiles (P5/P25/P50/P75/P95) |
| `annual_summary_bootstrap.csv` | Annual average forecast percentiles |
| `sims_freq_bootstrap.npy`, `sims_sev_bootstrap.npy` | Raw simulation draws, shape `(10000, 60)` |
| `fan_chart_bootstrap_comparison.png` | Forecast fan chart, fixed-parameter vs. bootstrap-augmented |
| `qq_innovations.png` | Q-Q diagnostic plots for the Gaussian-shock assumption |
| `empirical_distributions.png`, `empirical_cdf.png` | Distribution shape at future horizons |

## Requirements

```
python >= 3.9
numpy
pandas
matplotlib
scipy
```

Install with:
```bash
pip install numpy pandas matplotlib scipy
```

## Usage

```bash
python mc_bootstrap.py
```

Historical data is currently hardcoded at the top of the script. Replace the
`freq_pct` and `sev` lists (and adjust the `months` date range) with your own
monthly series to reforecast.

Key parameters you may want to change:
- `N_SIMS` (default 10,000) — number of Monte Carlo paths
- `N_BOOT` (default 500) — number of bootstrap parameter resamples
- `HORIZON` (default 60) — months forecast forward

## Methodology notes & known limitations

- **Log-linear trend is global.** A single trend line is fit across the full
  2020–2026 window. If the series has undergone a regime change (this dataset
  shows a sharp severity jump in Jan-2026), the trend and residual estimates
  will be distorted, and this leaks into the forecast.
- **Gaussian innovation assumption is violated.** The Q-Q plots and
  Shapiro-Wilk tests (run automatically) show heavy-tailed / outlier-driven
  innovations for both series — see `qq_innovations.png`. This means the
  simulated distributions are only approximately lognormal; tail percentiles
  (P5, P95) should be treated as indicative, not precise.
- **Correlation is Gaussian (via Cholesky).** Frequency/severity dependence is
  modeled as a single linear correlation coefficient. This will understate
  joint tail risk (both series spiking together in a systemic event) relative
  to a copula that allows tail dependence.
- **Parameter bootstrap uses a semi-parametric sieve method**, resampling
  AR(1) innovation pairs rather than assuming a parametric shock distribution.
  It still assumes AR(1) is the correct order and that `phi` doesn't itself
  vary over time.
- Only 84 monthly observations back the entire model — 7 seasonal cycles, and
  fewer effective degrees of freedom once trend, 11 seasonal dummies, and an
  AR(1) term are all estimated. Treat long-horizon (year 4–5) forecasts with
  appropriate caution.

## References

- Box, G.E.P., Jenkins, G.M., Reinsel, G.C. — *Time Series Analysis: Forecasting
  and Control*
- Cleveland, R.B. et al. (1990) — *STL: A Seasonal-Trend Decomposition
  Procedure Based on Loess*
- Klugman, S.A., Panjer, H.H., Willmot, G.E. — *Loss Models: From Data to
  Decisions* (standard actuarial reference for frequency/severity modeling)
- Bühlmann, P. (1997) — *Sieve Bootstrap for Time Series*, Bernoulli 3(2)
- England, P.D., Verrall, R.J. (2002) — *Stochastic Claims Reserving in
  General Insurance*
- Glasserman, P. — *Monte Carlo Methods in Financial Engineering*

## License

*(Add your preferred license — e.g. MIT — here.)*
