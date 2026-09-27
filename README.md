# Intraday Implied Volatility Surface Forecasting

A research project for forecasting **15-minute SPY implied total-variance surface dynamics**.

The project is designed around a simple idea:

> separate the mechanically predictable part of option-surface evolution from the genuinely stochastic market repricing component.

---

## Model

Rather than predicting the future volatility surface directly, the framework first removes deterministic maturity roll-down.

The 15-minute surface innovation is defined as:

$$
Y_t
=
W_{t+15}
-
\mathcal{R}_{15}(W_t)
$$

where:

- $W_t$ is the current implied total-variance surface,
- $\mathcal{R}_{15}$ is the 15-minute maturity roll-down operator.

A persistence-anchored convolutional neural network then predicts the conditional innovation.

The selected deterministic forecast is:

$$
\hat{W}_{t+15}
=
\mathcal{R}_{15}(W_t)
+
0.765\,f_\theta(H_t)
$$

where $H_t$ contains recent volatility surfaces, support masks, and market-state variables.

The coefficient $0.765$ is a validation-selected shrinkage factor applied to the neural correction.

---

## Surface Representation

The model works with implied total variance:

$$
w(k,\tau)
=
\tau \sigma_{\mathrm{IV}}^2
$$

where:

- $k = \log(K/S)$ is log-moneyness,
- $\tau$ is time to maturity,
- $\sigma_{\mathrm{IV}}$ is implied volatility.

Each surface is represented on a fixed grid of:

- **41 log-moneyness points**
- **10 maturity points**
- **410 total grid locations**

A support mask is maintained alongside every surface so that unsupported regions are not treated as observed data.

---

## Data Processing

The raw option data pipeline includes:

- bid/ask consistency checks,
- implied-volatility validity filters,
- stale-quote filtering,
- relative-spread filtering,
- DTE filtering,
- moneyness filtering,
- OTM option selection,
- irregular-strike interpolation,
- maturity interpolation,
- explicit support masks,
- no moneyness extrapolation outside observed support.

The resulting surfaces are constructed from intraday SPY option quotes sampled at 15-minute intervals.

Raw market data are intentionally not included in this repository.

---

## Forecasting Dataset

Each forecasting sample uses four historical surfaces:

$$
W_{t-45},
\quad
W_{t-30},
\quad
W_{t-15},
\quad
W_t
$$

and predicts the next 15-minute surface innovation.

The model input contains:

- 4 normalized historical surfaces,
- 4 historical support masks,
- current 15-minute SPY return,
- 1-hour realized volatility,
- intraday time encoding,
- log-moneyness coordinate,
- log-maturity coordinate.

This produces **14 input channels**.

Only strictly contiguous 15-minute windows are included.

---

## Persistence Baseline

The baseline forecast assumes no additional market repricing beyond maturity roll-down:

$$
\hat{W}_{t+15}^{\mathrm{pers}}
=
\mathcal{R}_{15}(W_t)
$$

Equivalently, the predicted innovation is:

$$
\hat{Y}_t^{\mathrm{pers}}
=
0
$$

This provides a strong short-horizon benchmark because implied-volatility surfaces are highly persistent over short intraday intervals.

---

## Neural Mean Model

The selected neural architecture is a smaller persistence-anchored CNN:

- 32 hidden channels,
- 2 residual blocks,
- 41,345 trainable parameters.

The model is initialized so that its raw innovation prediction is exactly zero.

Therefore, at initialization:

$$
\hat{W}_{t+15}
=
\mathcal{R}_{15}(W_t)
$$

and the network learns only deviations from the persistence baseline.

A larger model and a smaller model were also tested as capacity ablations.

---

## Model Capacity Ablation

| Model | Parameters | Test MSE Improvement vs Persistence | Test Day Win Rate |
|---|---:|---:|---:|
| V1 CNN | 304,641 | 7.23% | 51.7% |
| V2 Smaller CNN | 41,345 | 7.84% | 52.6% |
| V3 Tiny CNN | 6,753 | 6.46% | 48.7% |

The intermediate-capacity V2 model provided the best balance between model complexity and out-of-sample performance.

---

## Shrinkage Calibration

The raw CNN correction was found to be slightly too aggressive in many low-movement regimes.

A validation-selected constant shrinkage factor was therefore applied:

$$
\hat{Y}_t
=
0.765\,f_\theta(H_t)
$$

The final deterministic forecast becomes:

$$
\hat{W}_{t+15}
=
\mathcal{R}_{15}(W_t)
+
0.765\,f_\theta(H_t)
$$

A state-dependent shrinkage model was also tested, but its incremental MSE benefit over constant shrinkage was small. The constant specification was retained for parsimony.

---

## Current Out-of-Sample Results

On the held-out test period:

| Metric | Persistence | Final Model | Relative Change |
|---|---:|---:|---:|
| MSE | 7.7601e-08 | 7.1712e-08 | **7.59% lower** |
| RMSE | 2.7857e-04 | 2.678e-04 | approximately **3.9% lower** |
| MAE | 8.5971e-05 | approximately 8.95e-05 | approximately **4.1% higher** |
| Day-level win rate | — | **60.4%** | — |

The model therefore improves squared-error performance relative to roll-down persistence, although persistence remains difficult to beat in many low-movement individual samples.

The strongest gains occur when persistence makes relatively large errors.

---

## Evaluation Methodology

The dataset is split chronologically into:

- training,
- validation,
- test.

Model architecture, early stopping, and shrinkage parameters are selected using validation data.

Evaluation includes:

- raw-space MSE,
- RMSE,
- MAE,
- sample-level win rates,
- trading-day win rates,
- day-level bootstrap confidence intervals,
- clock-time analysis,
- realized-volatility regimes,
- persistence-error regimes.

---

## Repository Structure

The main notebook is:

```text
intraday_iv_surface_forecasting.ipynb
```

Supporting files:

```text
README.md
requirements.txt
.gitignore
```

Raw market data and trained model checkpoints are not included.

---

## Data Setup

Set the environment variable `SPY_OPTION_DATA_ROOT` to the directory containing the raw SPY option files.

### Windows PowerShell

```powershell
$env:SPY_OPTION_DATA_ROOT="D:\path\to\SPY"
```

### macOS / Linux

```bash
export SPY_OPTION_DATA_ROOT=/path/to/SPY
```

The expected raw-data layout contains one subdirectory per trading date.

---

## Installation

Create a Python environment and install the required dependencies:

```bash
pip install -r requirements.txt
```

Main dependencies include:

- NumPy
- pandas
- PyTorch
- scikit-learn
- matplotlib
- tqdm

---

## Research Direction

The current model is a one-step 15-minute conditional-mean forecast.

Planned extensions include:

1. a more complete transport baseline incorporating spot-induced moneyness movement,
2. low-rank conditional-mean forecasting,
3. residual covariance and PCA analysis,
4. effective-rank estimation,
5. latent Gaussian residual baselines,
6. conditional Flow Matching for low-dimensional residual distributions,
7. multi-step volatility-surface trajectory generation.

The long-term objective is to decompose future surface dynamics into:

$$
\boxed{
\text{mechanical transport}
+
\text{predictable conditional mean}
+
\text{stochastic residual dynamics}
}
$$
