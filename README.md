# Energy Demand Forecasting & Grid Planner

Short-horizon **hourly electricity demand forecasting** for the ERCOT (Texas) power grid,
built as an end-to-end data-science pipeline and shipped as a live interactive web app.

**Live app:** https://ercot-grid-planner1.hub.zerve.cloud

---

## The Problem

Electricity cannot be stored cheaply at grid scale, so supply must match demand in near
real time. Grid operators commit generators hours ahead based on a demand forecast, and a
wrong forecast is expensive in both directions:

- **Under-forecast** → too little generation committed → costly emergency peaker plants,
  and in extreme cases load-shedding (blackouts).
- **Over-forecast** → too much generation committed → wasted fuel and curtailment cost.

Demand is driven mainly by **weather** (air-conditioning in Texas summers, heating in cold
snaps) and **calendar patterns** (hour of day, weekday vs. weekend, holidays). The goal:
forecast hourly demand from recent load, weather, and the calendar — and **quantify the
uncertainty** so an operator can size a reserve margin instead of trusting a single number.

## Results

| Metric | Value |
| --- | --- |
| Walk-forward backtest MAPE | **2.67%** |
| Improvement over seasonal-naive baseline (8.70%) | **69%** |
| Prediction-interval coverage (80% target) | **82.5%** empirical |

- Evaluation uses a **walk-forward backtest** (train on past, predict the next period, roll
  forward) — not a random split, which would leak future information into the past.
- Uncertainty is calibrated with **Conformalized Quantile Regression (CQR)**, giving honest
  intervals rather than over-confident ones.

## The Data

Two live, free, public sources:

| Source | What | Why |
| --- | --- | --- |
| [EIA Open Data API v2](https://www.eia.gov/opendata/) | ~2 years of hourly ERCOT demand (MW), respondent `ERCO`, type `D` | Authoritative government load data; updated continuously |
| [Open-Meteo Archive API](https://open-meteo.com/) | Hourly temperature / humidity / wind for Dallas (ERA5) | Keyless; demand is largely a function of weather |

Both are pulled with a **dynamic window** (up to the most recent available day), so the
project stays current instead of being a frozen snapshot. EIA timestamps are UTC and are
standardized on ingestion, then localized to `America/Chicago` for calendar features.

## The Pipeline

Built as a connected block DAG on the [Zerve](https://www.zerve.ai/) canvas, mirrored here
as a single self-contained app:

```
A1  Ingest ERCOT demand (EIA, paginated)
A2  Ingest Dallas weather (Open-Meteo)            ── parallel
B   SQL join (DuckDB) + calendar features
C   Clean + data-quality report
D   EDA (seasonality, temperature–demand curve)
E   Feature engineering (leakage-safe)
F1  Baseline: seasonal-naive       F2  Gradient-boosted model   ── parallel
G   Walk-forward backtest + metrics + CQR intervals
H   Grid Planner app (deployed Streamlit)
```

**Features** (leakage-safe — only past information at prediction time):

- Lagged demand: 24h, 48h, 168h (one week ago)
- Rolling mean/std of demand, `shift(1)` so the current hour never predicts itself
- Heating / cooling degree-days around an 18 °C comfort base
- Cyclical sin/cos encodings of hour-of-day and day-of-year
- Calendar flags: weekend, US federal holiday

**Model:** `HistGradientBoostingRegressor` quantile models (10th / 50th / 90th percentile),
with a held-out calibration set computing the conformal correction `Q`.

## The Grid Planner App

An interactive tool that turns the pipeline into something an operator (or a recruiter) can
use without writing code:

- **Pick any day** and see the forecast curve with the calibrated **80% confidence band**.
- **Temperature scenario slider** — shift the day warmer/colder and watch the forecast and
  peak respond, the way an operator stress-tests a weather outlook.
- **Key metrics:** predicted peak demand, the hour it occurs, and a worst-case (upper-band)
  reserve figure.
- **Operator recommendation** — a tiered, plain-English readout (light / typical / elevated
  / extreme) based on where the forecast sits against historical demand.

## Tech Stack

Python · pandas / NumPy · DuckDB (SQL) · scikit-learn (HistGradientBoosting) · Plotly ·
Streamlit · Zerve (canvas + deployment) · EIA & Open-Meteo APIs

## Run It Locally

```bash
pip install -r requirements.txt
export EIA_API_KEY="your_free_key"   # register at https://www.eia.gov/opendata/register.php
streamlit run app/main.py
```

The first load fetches ~2 years of data and trains the models (cached afterward), so it
takes a minute; subsequent interactions are instant.

> The EIA API key is read from the `EIA_API_KEY` environment variable (or a Streamlit
> secret) and is never committed to the repository.

## Repository Layout

```
ercot-energy-demand-forecasting/
├── app/
│   └── main.py          # self-contained pipeline + interactive Grid Planner
├── requirements.txt
├── .gitignore
├── LICENSE
└── README.md
```
