# AQI Analysis and Next-Day Forecasting for 7 Indian Cities

Trend analysis of air quality across 7 Indian cities, benchmarking of Delhi's pollutant levels
against CPCB national limits, and next-day AQI forecasting with lag features, XGBoost and an
artificial neural network (ANN).

> **About this repository.** This is a re-implementation of the analysis from my research
> internship at IIT Bhubaneswar (June–July 2026, under Dr. Rajesh Roshan Dash). The original
> code and data remained on the lab's systems, so I rebuilt the pipeline from scratch on the
> public CPCB-derived dataset. Numbers here come from this rebuild and are not identical to the
> original internship results.

## Why it matters

AQI forecasts give a day's warning before pollution episodes. Authorities can act under Delhi's
Graded Response Action Plan (GRAP) before a spike rather than after it, health advisories can go
out in advance, and hospitals can plan for respiratory admissions. The same lag-feature approach
applies to any daily demand-forecasting problem.

## Data

- Source: CPCB monitoring data, daily city level, 2015-01-01 to 2020-07-01 ([details](data/README.md)).
- Cities: Ahmedabad, Bengaluru, Chennai, Delhi, Hyderabad, Lucknow, Mumbai.
- Size: 14,060 city-days.
- Pollutants used: PM2.5, PM10, NO2, SO2, CO, O3, NH3.

## Pipeline

Four notebooks in `notebooks/`, run in order. Each one is self-contained (its own config cell)
and passes data to the next through `data/processed/`.

```
data/city_day.csv
  -> 01_data_preparation.ipynb       filter 7 cities, AQI capped at 500 (the scale maximum),
                                     missing-value report, continuous daily calendar per city,
                                     negative values -> NaN, past-only forward fill (max 3 days)
                                     => data/processed/clean.csv
  -> 02_eda_cpcb_benchmarking.ipynb  60-day trends across cities; Delhi vs CPCB NAAQS limits
                                     (60-day window and 2015-2020)
  -> 03_feature_engineering.ipynb    lags (t-1, t-2, t-3, t-7), rolling mean/std of past AQI,
                                     yesterday's pollutants, calendar features, city one-hot;
                                     forecast and leaky "nowcast" variants
                                     => data/processed/features_{forecast,nowcast}.csv
  -> 04_modeling_and_evaluation.ipynb  chronological split, persistence baseline, Linear
                                     Regression, XGBoost, ANN; RMSE / MAE / R2 / AQI-category
                                     accuracy; feature importance; leakage check
```

### Design decisions

- **One-day-ahead forecasting only.** Every feature for day *t* uses information available by
  the end of day *t-1*. AQI is computed from same-day pollutant concentrations, so using them as
  inputs would let the model reverse-engineer the AQI formula instead of forecasting.
  Notebook 04 runs that leaky "nowcast" variant separately to show the gap.
- **Chronological split, no shuffling.** Train: 2015 to Jun 2018. Validation (early stopping
  only): Jul to Dec 2018. Test: Jan 2019 to Jul 2020.
- **Lags computed inside each city's series**, so one city's history never leaks into another's.
- **Target never imputed.** Days without a measured AQI are dropped from training and testing.
  Inputs are forward-filled from the past only. Linear interpolation would use future values.
- **Preprocessing fit on the training period only**: median imputer and scalers for Linear
  Regression and the ANN. XGBoost handles missing values natively.
- **Data-quality rule.** Ahmedabad's CO readings are physically implausible (median about
  16 mg/m3 against about 1.2 in Delhi), which pushes its reported AQI to 2,049. AQI is capped at
  500, and results are also reported with Ahmedabad excluded.

## Results

### 60-day study window (1 Nov to 30 Dec 2019)

| City | Mean AQI | % days Poor or worse (>200) |
|---|---|---|
| Delhi | 346 | 90.0 |
| Ahmedabad* | 335 | 80.0 |
| Lucknow | 310 | 88.3 |
| Mumbai | 160 | 21.7 |
| Hyderabad | 135 | 0.0 |
| Chennai | 101 | 11.7 |
| Bengaluru | 89 | 0.0 |

\*Ahmedabad's values are inflated by suspect CO readings.

**Delhi vs CPCB NAAQS (24-hour limits), same window:** PM2.5 averaged 3.4x its limit and
exceeded it on 96.7% of days. PM10 averaged 3.1x its limit and exceeded it on 98.3% of days.
Every other pollutant averaged below its limit. Particulate matter is the driver of Delhi's AQI.
Over 2015–2020 the figures are PM10 85.9% and PM2.5 72.5% of days above the limit.

### Next-day AQI forecast, test period Jan 2019 to Jul 2020

| Model | RMSE (7 cities) | RMSE (excl. Ahmedabad) | MAE (excl.) | R² (excl.) |
|---|---|---|---|---|
| Persistence baseline (AQI_t = AQI_t-1) | 41.9 | 31.8 | 19.5 | 0.858 |
| Linear Regression | 37.9 | 28.0 | 18.7 | 0.891 |
| **XGBoost** | **33.5** | **25.5** | **16.8** | **0.909** |
| ANN (128-64-32, ReLU, dropout, Adam, early stopping) | 37.2 | 29.9 | 22.9 | 0.875 |

- XGBoost beats the persistence baseline by about 20% in RMSE (excl. Ahmedabad).
- On this public dataset gradient boosting outperformed the ANN. That is common on mid-sized
  tabular data.
- **Leakage check:** adding same-day pollutants as inputs drops XGBoost's RMSE to 22.1
  (R² 0.967). That setup is not a forecast, because those values are not known in advance.

Per-city and per-period breakdowns are in `results/`.

Figures in `results/figures/`: `aqi_trends_60_days.png`, `delhi_vs_naaqs.png`,
`delhi_actual_vs_pred.png`, `xgb_feature_importance.png`.

## Run it

```bash
python -m venv venv && source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

Then open the notebooks in Jupyter (`jupyter lab`) and run `01` → `02` → `03` → `04` in order,
or execute them all from the command line:

```bash
jupyter nbconvert --to notebook --execute --inplace notebooks/0*.ipynb
```

Runtime is a few minutes on CPU. All tables and figures are regenerated in `results/`
(figures are also shown inline); intermediate data goes to `data/processed/`.

## Limitations and next steps

- No meteorological inputs (wind speed, temperature, humidity, boundary-layer height), which
  drive winter pollution episodes in north India.
- One fixed train/test split. Rolling-origin cross-validation would give more reliable comparisons.
- The 2020 test period includes the COVID-19 lockdown, a distribution shift.
- Next steps: weather features, `TimeSeriesSplit` cross-validation, per-city models, SHAP
  explanations, and a classification version ("will tomorrow be Poor or worse?").

## Project structure

```
├── data/
│   ├── city_day.csv
│   ├── README.md
│   └── processed/                          generated intermediate data (git-ignored)
├── notebooks/
│   ├── 01_data_preparation.ipynb           loading, cleaning, data-quality rules
│   ├── 02_eda_cpcb_benchmarking.ipynb      60-day trends and NAAQS benchmarking
│   ├── 03_feature_engineering.ipynb        lag, rolling, calendar and city features
│   └── 04_modeling_and_evaluation.ipynb    models, evaluation, leakage check
├── results/                                generated tables and figures
└── requirements.txt
```
