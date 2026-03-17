# TTF Gas Forward Curve — PCA & Monte Carlo Simulation

## Overview

This notebook performs Principal Component Analysis (PCA) on TTF natural gas forward curve price data and uses the results to run a Monte Carlo simulation of future curve scenarios.

**Input:** `quotes.xlsx` — daily TTF forward prices across up to 25 tenors (spot, monthly, quarterly, seasonal, calendar year)
**Output:** `mc_simulations.xlsx` — one simulated 10-year daily price path in the same format as the input

---

## Notebook structure

| Cell | Description |
|---|---|
| Config | File paths, column range, skip rows |
| Data loading | Read Excel, forward-fill gaps, compute log-returns |
| PCA | Standardise returns, fit PCA, build eigenvalue / variance summary |
| Scree plot | Eigenvalue bar chart with Kaiser criterion |
| Cumulative variance | Elbow chart; prints Level / Slope / Curvature shares |
| Loadings | Bar charts for PC1–PC3 per tenor |
| **Design notes** | Why naive MC approaches fail; correct methodology (see below) |
| MC fan chart | 500-path simulation, 30-day horizon, percentile bands |
| MC export | 1 path × 10 years written to `mc_simulations.xlsx` |
| Consistency check | KS tests, vol comparison, correlation heatmaps, PC1–PC5 score distributions |

---

## Monte Carlo Simulation — Design Notes

### Why the naive approaches failed

#### Attempt 1 — Gaussian shocks on PC scores

The first implementation drew daily factor shocks from a normal distribution:

```python
shocks = rng.standard_normal(N_PC) * sqrt(eigenvalue_i)   # N(0, λ_i)
```

**Why it fails the KS test:**
Energy market log-returns are **not Gaussian**. They exhibit:
- **Fat tails** — extreme moves (e.g. gas spikes) occur far more often than a normal distribution predicts
- **Skewness** — upside spikes are sharper and larger than downside moves
- **Zero-return clustering** — many days with identical prices produce a mass of zero log-returns

A Gaussian draw discards all of this empirical structure. The KS test rejected every tenor because the simulated tails were far too thin.

---

#### Attempt 2 — Bootstrap of PC scores (3 components)

The second approach resampled whole rows from the historical PC score matrix instead of drawing from a normal distribution:

```python
shock = hist_scores[idx, :N_PC]          # resample one historical day
z_shock = shock @ pca.components_[:N_PC] # reconstruct standardised log-return
log_ret = z_shock * scaler.scale_ + scaler.mean_
```

Using only **N_PC = 3** components means the reconstruction captures ~91 % of variance.
**Why it still fails:**
The 9 % of variance carried by PC4–PC_n is silently discarded. Each simulated return vector is a projection onto a 3-dimensional subspace, so the marginal distribution of every tenor is slightly compressed relative to history. With ~2 700 observations the two-sample KS test has enough power to detect even this small systematic difference.

---

#### Attempt 3 — Bootstrap of PC scores (all components)

Extending to all `n_components` should in theory give exact reconstruction:

```python
z_reconstructed = hist_scores[idx, :] @ pca.components_   # all PCs
```

**Why it still fails:**
Even though the numerical reconstruction error is ~10⁻¹⁵, routing through the PCA transform/inverse introduces floating-point artefacts that alter the empirical CDF in ways the high-power KS test detects (observed: 1/23 tenors passing vs 23/23 for direct bootstrap with the same random indices).

---

### Correct approach — Direct bootstrap of historical return vectors

The fix is to skip the PCA reconstruction entirely during simulation and resample the original log-return rows directly:

```python
idx   = rng.integers(0, len(log_ret_arr))
curve = curve * np.exp(log_ret_arr[idx])   # one full historical day
```

#### Why this is correct

| Property | Guaranteed by direct bootstrap |
|---|---|
| Marginal distribution per tenor | Exact — sampling from the empirical CDF |
| Cross-tenor correlation | Exact — whole rows are drawn jointly |
| Fat tails & skewness | Exact — no distributional assumption |
| Zero-return days | Preserved — zero rows reappear with historical frequency |
| KS test | Passes all tenors (observed: 23/23) |

#### Relationship to PCA

PCA is still the right tool for **analysis** — it decomposes the curve's risk into interpretable factors (Level, Slope, Curvature) and tells us how much variance each carries.
For **simulation**, however, bootstrapping the original return vectors is equivalent to resampling all PC scores jointly and reconstructing with all components — just without the numerical roundtrip that corrupts the distribution.

#### Limitations to be aware of

- **No out-of-sample scenarios** — only moves that occurred historically can be simulated; the model cannot generate a move larger than the historical maximum.
- **No time-series structure** — each day is drawn i.i.d.; volatility clustering (GARCH effects) is not captured.
- **Stationarity assumption** — the simulation implicitly assumes the future return distribution is the same as the historical window used (`skip` rows back from today).

---

## Requirements

```
numpy
pandas
matplotlib
scikit-learn
scipy
openpyxl
```
