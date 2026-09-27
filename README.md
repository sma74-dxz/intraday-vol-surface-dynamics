# Intraday Implied Volatility Surface Forecasting

Research project for forecasting 15-minute SPY implied total-variance surface dynamics.

## Model

Rather than predicting the future surface directly, the project separates
deterministic maturity roll-down from the surface innovation:

\[
Y_t = W_{t+15} - \mathcal{R}_{15}(W_t).
\]

A persistence-anchored convolutional neural network predicts the conditional
innovation.  The selected deterministic forecast is

\[
\hat W_{t+15}
=
\mathcal{R}_{15}(W_t)
+
0.765 f_\theta(H_t).
\]

The input history contains four 15-minute surfaces together with their support
masks and market-state features.

## Surface representation

- SPY listed options
- 15-minute intraday observations
- implied total variance \(w=\tau\sigma_{\mathrm{IV}}^2\)
- 41 log-moneyness points
- 10 maturity points
- explicit support masks
- no moneyness extrapolation outside observed support
- stale-quote and relative-spread filtering

## Current result

On the held-out test period, the validation-selected V2 + constant-shrinkage
specification reduces MSE by approximately **7.59%** relative to roll-down
persistence.  The model wins on approximately **60.4% of test trading days**.

These results are experimental and are reported in total-variance space.

## Repository

The main notebook is:

`intraday_iv_surface_forecasting.ipynb`

Raw market data are intentionally not included.

## Data setup

Set an environment variable pointing to the local directory containing raw SPY
option files:

### Windows PowerShell

```powershell
$env:SPY_OPTION_DATA_ROOT="D:\path\to\SPY"
```

### macOS / Linux

```bash
export SPY_OPTION_DATA_ROOT=/path/to/SPY
```

The expected layout is one directory per trading date.

## Reproducibility

The notebook uses chronological train/validation/test splits.  Model selection
and shrinkage parameters are selected using validation data; the test period is
used for out-of-sample evaluation.

## Research roadmap

Planned extensions include a more complete transport baseline, low-rank
conditional-mean forecasting, residual PCA/effective-rank analysis, and
conditional generative modeling of low-dimensional residual dynamics.
