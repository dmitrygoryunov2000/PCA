# TTF Natural Gas Forward Curve — PCA Analysis & Monte Carlo Simulation

> Principal Component Analysis of TTF natural gas forward curves with Monte Carlo simulation and rolling stability analysis.

---

## Table of Contents

- [Overview](#overview)
- [Dataset](#dataset)
- [Notebook Structure](#notebook-structure)
- [Analyses Performed](#analyses-performed)
  - [1. Full PCA Analysis](#1-full-pca-analysis)
  - [2. Spread-Inclusive PCA](#2-spread-inclusive-pca-summerwinter-spreads)
  - [3. Monte Carlo Simulation](#3-monte-carlo-simulation)
  - [4. Rolling PCA — Factor Stability](#4-rolling-pca--factor-stability)
- [Monte Carlo Simulation — Design Notes](#monte-carlo-simulation--design-notes)
- [Key Findings](#key-findings)
- [Repository Structure](#repository-structure)
- [Setup & Installation](#setup--installation)
- [Usage](#usage)
- [License](#license)

---

## Overview

This project applies **Principal Component Analysis (PCA)** to the TTF (Title Transfer Facility) natural gas forward curve to decompose price dynamics into interpretable risk factors.

**Input:** `quotes.xlsx` — daily TTF forward prices across up to 25 tenors (spot, monthly, quarterly, seasonal, calendar year)
**Output:** `mc_simulations.xlsx` — one simulated 10-year daily price path in the same format as the input

The analysis covers:

1. **Full PCA** on 25 forward price tenors
2. **Spread-inclusive PCA** incorporating summer-winter seasonality
3. **Monte Carlo simulation** of forward curves using bootstrapped historical returns
4. **Rolling PCA** to assess factor stability across market regimes (2015–2025)

The TTF is the most liquid European natural gas benchmark, and understanding its forward curve dynamics is critical for trading, hedging, and risk management.

---

## Dataset

| Property | Detail |
|----------|--------|
| **Source** | TTF forward price quotes |
| **File** | `quotes.xlsx` (Sheet1) |
| **Rows** | 4,654 trading days |
| **Columns** | 25 tenors + Date index |
| **Period** | ~2006–2025 |
| **Usable observations** | 2,690 (after removing NaN/zero rows for log returns) |

### Tenor Structure (25 contracts)

| Category | Tenors | Count |
|----------|--------|-------|
| Spot / Near-term | DAYSPOT, WDNW (Weekend), MTBAL (Month Balance) | 3 |
| Month-Ahead | MM, M+1, M+2, M+3, M+4, M+5 | 6 |
| Quarters | QTR01 – QTR06 | 6 |
| Seasons | SEA01 – SEA06 (alternating Summer/Winter) | 6 |
| Calendar Years | YRC2, YRC3, YRC4, YRG1 | 4 |

---

## Notebook Structure

| Cell | Description |
|------|-------------|
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

## Analyses Performed

### 1. Full PCA Analysis

**Sheet**: `PCA_Results`
**Method**: Eigendecomposition of the correlation matrix of standardised log returns
**Observations**: 2,690 × 25 tenors

#### Eigenvalues & Variance Explained

| Component | Eigenvalue | Variance % | Cumulative % | Interpretation |
|-----------|-----------|-----------|-------------|----------------|
| **PC1** | 16.687 | 66.75% | 66.75% | **Level** — parallel shift |
| **PC2** | 2.243 | 8.97% | 75.72% | **Slope** — front vs. back tilt |
| **PC3** | 1.537 | 6.15% | 81.87% | **Curvature** — butterfly/belly |
| PC4 | 0.968 | 3.87% | 85.74% | |
| PC5 | 0.801 | 3.20% | 88.95% | |
| PC6–PC25 | <0.59 each | 11.05% | 100.00% | |

#### Factor Interpretation

- **PC1 (Level, 66.75%)**: All tenors load uniformly positive (0.17–0.23). Represents a parallel shift of the entire forward curve — the dominant risk factor.
- **PC2 (Slope, 8.97%)**: Front-end loads negative (−0.21), back-end loads positive (+0.33–0.39). Captures the tilt/steepening of the curve.
- **PC3 (Curvature, 6.15%)**: Front and back load same sign, middle loads opposite. Classic butterfly pattern — the belly moves independently.
- **Kaiser criterion**: Only PC1–PC3 have eigenvalues > 1. Three factors are sufficient.

---

### 2. Spread-Inclusive PCA (Summer/Winter Spreads)

**Sheet**: `PCA_SW_Spreads`
**Modification**: Replaced 3 winter season contracts (SEA02, SEA04, SEA06) with summer-winter spread ratios: `log(Winter / Summer)` returns
**Variables**: 22 original tenors + 3 S-W spread ratios = 25

#### Results

| Component | Eigenvalue | Variance % | Cumulative % | Interpretation |
|-----------|-----------|-----------|-------------|----------------|
| **PC1** | 15.326 | 61.3% | 61.3% | **Level** — spreads load near zero |
| **PC2** | 3.209 | 12.8% | 74.1% | **Seasonality** — all 3 spreads load +0.40 to +0.49 |
| **PC3** | 1.869 | 7.5% | 81.6% | **Slope** — near-term spread loads +0.30 |
| PC6 | 0.602 | 2.4% | — | **Spread curve** — near vs. far spread inversion |

#### Comparison with Standard PCA

| Metric | Original PCA | Spread PCA |
|--------|-------------|-----------|
| PC1 Variance | 66.75% | 61.3% |
| PC2 Variance | 8.97% | 12.84% |
| PC3 Variance | 6.15% | 7.48% |
| Top 3 Cumulative | 81.87% | 81.61% |
| PC2 Interpretation | Slope (front vs back) | **Seasonality** (S-W dynamics) |
| Spread visibility | Implicit | **Explicit** — dedicated factor |

**Insight**: Summer-winter seasonality is an **independent risk factor** (~13% of variance) that is hidden in the standard PCA but explicitly captured when spreads are included.

---

### 3. Monte Carlo Simulation

**Sheet**: `MC_Simulation`
**Model**: Direct bootstrap of historical return vectors
**Scenarios**: 1,000 simulated 1-day forward curves
**Variance captured**: 100% (full empirical distribution)

#### Methodology

```
Step 1: Compute log-return matrix from historical prices
Step 2: Sample one full row (all 25 tenors) uniformly at random
Step 3: Apply to current curve: P_new = P_current * exp(log_return_row)
Step 4: Repeat for each simulation step
```

See [Monte Carlo Simulation — Design Notes](#monte-carlo-simulation--design-notes) for why earlier PCA-based approaches were superseded by direct bootstrapping.

#### Selected VaR Results

| Tenor | Current Price (EUR) | Std Dev | 95% VaR (EUR) | 99% VaR (EUR) | VaR/Price |
|-------|-------------------|---------|-------------|-------------|-----------|
| DAYSPOT | 30.34 | 1.46 | 2.38 | 3.07 | 7.8% |
| WDNW | 30.40 | 1.38 | 2.21 | 2.87 | 7.3% |
| MTBAL | 30.85 | 1.35 | 2.16 | 2.80 | 7.0% |
| QTR01 | 31.34 | 1.20 | 1.90 | 2.59 | 6.1% |
| SEA01 | 29.76 | 1.13 | 1.88 | 2.54 | 6.3% |
| YRC2 | 28.46 | 0.61 | 1.00 | 1.39 | 3.5% |
| YRC3 | 25.70 | 0.47 | 0.77 | 1.05 | 3.0% |
| YRC4 | 24.05 | 0.32 | 0.52 | 0.72 | **2.2%** |

**Finding**: Front-end tenors have 3-4x higher relative VaR than back-end contracts.

---

### 4. Rolling PCA — Factor Stability

**Sheet**: `Rolling_PCA`
**Window**: 504 trading days (~2 years)
**Step**: 126 trading days (~6 months)
**Windows**: 18 rolling periods spanning 2015–2025

#### Variance Explained Over Time

| Window End | PC1 (Level) | PC2 (Slope) | PC3 (Curvature) | Top 3 |
|------------|------------|------------|-----------------|-------|
| 2017-06 | 77.5% | 7.8% | 5.3% | 90.6% |
| 2018-11 | 61.6% | 14.0% | 6.7% | 82.3% |
| **2020-11** | **55.7%** | **13.7%** | **11.1%** | **80.5%** |
| 2022-10 | 69.8% | 12.9% | 5.6% | 88.2% |
| 2025-03 | 79.0% | 6.1% | 4.4% | 89.5% |
| 2025-09 | 78.8% | 6.3% | 5.8% | 90.9% |

#### Regime Analysis

- **Calm markets (2017, 2025)**: PC1 dominates at ~78%, simple level-driven dynamics, top 3 ~90%
- **Pre-crisis (2018–2019)**: PC1 drops to ~60%, more complex multi-factor dynamics
- **Energy crisis (2020–2022)**: PC1 at minimum (55.7%), curvature rises to 11.1%, risk diversifies across factors
- **Post-crisis (2023+)**: Recovery to level-dominated structure, but PC2 seasonal pattern permanently shifted

---

## Monte Carlo Simulation — Design Notes

PCA is the right tool for **analysis** — decomposing the curve's risk into Level, Slope, and Curvature factors. For **simulation**, however, three earlier PCA-based approaches were tested and rejected before arriving at the correct method.

### Why the Naive Approaches Failed

#### Attempt 1 — Gaussian Shocks on PC Scores

The first implementation drew daily factor shocks from a normal distribution:

```python
shocks = rng.standard_normal(N_PC) * sqrt(eigenvalue_i)   # N(0, λ_i)
```

**Why it fails the KS test:**
Energy market log-returns are **not Gaussian**. They exhibit fat tails (extreme moves far more frequent than normal predicts), skewness (upside spikes larger than downside), and zero-return clustering (many days with identical prices). A Gaussian draw discards all of this empirical structure. The KS test rejected every tenor because the simulated tails were far too thin.

---

#### Attempt 2 — Bootstrap of PC Scores (3 Components)

The second approach resampled whole rows from the historical PC score matrix:

```python
shock = hist_scores[idx, :N_PC]          # resample one historical day
z_shock = shock @ pca.components_[:N_PC] # reconstruct standardised log-return
log_ret = z_shock * scaler.scale_ + scaler.mean_
```

**Why it still fails:**
Using only **N_PC = 3** silently discards the 9% of variance carried by PC4–PCn. Each simulated return vector is a projection onto a 3-dimensional subspace, marginally compressing every tenor relative to history. With ~2,700 observations the KS test has enough power to detect even this small systematic difference.

---

#### Attempt 3 — Bootstrap of PC Scores (All Components)

Extending to all `n_components` should in theory give exact reconstruction:

```python
z_reconstructed = hist_scores[idx, :] @ pca.components_   # all PCs
```

**Why it still fails:**
Even though the numerical reconstruction error is ~10⁻¹⁵, routing through the PCA transform/inverse introduces floating-point artefacts that alter the empirical CDF in ways the high-power KS test detects (observed: 1/23 tenors passing vs 23/23 for direct bootstrap with the same random indices).

---

### Correct Approach — Direct Bootstrap of Historical Return Vectors

The fix is to skip PCA reconstruction entirely during simulation and resample the original log-return rows directly:

```python
idx   = rng.integers(0, len(log_ret_arr))
curve = curve * np.exp(log_ret_arr[idx])   # one full historical day
```

#### Why This Is Correct

| Property | Guaranteed by Direct Bootstrap |
|----------|-------------------------------|
| Marginal distribution per tenor | Exact — sampling from the empirical CDF |
| Cross-tenor correlation | Exact — whole rows are drawn jointly |
| Fat tails & skewness | Exact — no distributional assumption |
| Zero-return days | Preserved — zero rows reappear with historical frequency |
| KS test | Passes all tenors (observed: 23/23) |

This is equivalent to resampling all PC scores jointly and reconstructing with all components — just without the numerical roundtrip that corrupts the distribution.

#### Limitations

- **No out-of-sample scenarios** — only moves that occurred historically can be simulated; the model cannot generate a move larger than the historical maximum.
- **No time-series structure** — each day is drawn i.i.d.; volatility clustering (GARCH effects) is not captured.
- **Stationarity assumption** — the simulation implicitly assumes the future return distribution matches the historical window used.

---

## Key Findings

1. **Three-Factor Model Is Sufficient** — Level (67%), Slope (9%), Curvature (6%) explain ~82% of variance, consistent with classic fixed-income term structure theory applied to energy markets.

2. **Level Is Regime-Dependent** — PC1 variance share ranged from 55.7% (Nov 2020, crisis) to 79.0% (Mar 2025, calm). In crises, risk diversifies to slope and curvature.

3. **Seasonality Is an Independent Risk Factor** — Summer-winter seasonality accounts for ~13% of variance when explicitly extracted via spread ratios. It is orthogonal to the level factor.

4. **Front-End Carries Most Risk** — DAYSPOT 95% VaR = EUR 2.38 (7.8% of price) vs. YRC4 = EUR 0.52 (2.2%). Front-end tenors are **3-4x more volatile** than back-end contracts.

5. **Energy Crisis Was a Structural Break** — The 2020–2022 period required more factors (PC3 rose to 11%), and PC2's seasonal loading pattern shifted permanently.

6. **DAYSPOT as a Canary** — DAYSPOT's loading on PC1 ranged 0.06–0.22 across windows. When spot decouples from the curve (storage/weather events), its level loading drops sharply — a natural indicator of spot-forward basis risk.

---

## Repository Structure

```
ttf-pca-analysis/
│
├── README.md                      # This file
├── requirements.txt               # Python dependencies
├── .gitignore                     # Git ignore rules
│
├── data/
│   └── quotes.csv                 # Raw TTF forward prices (export from Sheet1)
│
├── src/
│   ├── config.py                  # Configuration & parameters
│   └── pca_forward_curve.py       # Main PCA analysis script
│
├── notebooks/
│   └── pca_analysis.ipynb         # Jupyter notebook (full analysis)
│
├── results/
│   ├── pca_results.csv            # Eigenvalues, loadings, variance explained
│   ├── pca_sw_spreads.csv         # Spread-inclusive PCA results
│   ├── mc_simulation.csv          # Monte Carlo VaR statistics
│   └── rolling_pca.csv            # Rolling window stability analysis
│
└── quotes.xlsx                    # Original Excel workbook (optional)
```

---

## Setup & Installation

### Prerequisites

- Python 3.9+
- pip or conda

### Installation

```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/ttf-pca-analysis.git
cd ttf-pca-analysis

# Create virtual environment (recommended)
python -m venv venv
source venv/bin/activate  # Linux/Mac
# venv\Scripts\activate   # Windows

# Install dependencies
pip install -r requirements.txt
```

### requirements.txt

```
numpy>=1.24
pandas>=2.0
scipy>=1.10
scikit-learn>=1.2
matplotlib>=3.7
seaborn>=0.12
openpyxl>=3.1
jupyter>=1.0
```

---

## Usage

### Run the PCA analysis

```bash
python src/pca_forward_curve.py
```

### Launch Jupyter notebook

```bash
jupyter notebook notebooks/pca_analysis.ipynb
```

### Export data from Excel

```python
import openpyxl, csv, os

wb = openpyxl.load_workbook("quotes.xlsx", data_only=True)
os.makedirs("results", exist_ok=True)

for name in wb.sheetnames:
    ws = wb[name]
    with open(f"results/{name}.csv", "w", newline="") as f:
        writer = csv.writer(f)
        for row in ws.iter_rows(values_only=True):
            writer.writerow(row)
```

---

## License

This project is provided for educational and research purposes. The underlying price data is proprietary and subject to the data provider's license terms.
