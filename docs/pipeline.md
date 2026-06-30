# Pipeline Narration

This is the stage-by-stage narration that accompanies the project on the
[Zerve](https://www.zerve.ai/) canvas, where the work is built as a connected block DAG.
Each section below sits above the corresponding block(s) on the canvas.

---

## Stage A — Data Ingestion

Two independent data sources are pulled in parallel, then joined downstream.

### A1 — Electricity demand (EIA Open Data API)

- Source: U.S. Energy Information Administration (EIA) v2 API,
  `electricity/rto/region-data`, respondent **ERCO** (ERCOT), type **D** (demand).
- ~2 years of hourly demand in MW, paginated 5,000 rows per request.
- The window is **dynamic** — it pulls up to yesterday on every run, so the project
  stays current instead of being frozen to a fixed snapshot.
- The API key is stored as a **Zerve Secret**, not hard-coded.

### A2 — Weather (Open-Meteo Archive API)

- Hourly temperature for the Dallas region (lat 32.78, lon -96.80), the demand center
  of ERCOT.
- Keyless ERA5 reanalysis archive; it trails real time by ~5 days, so the window ends
  at **now − 6 days**.

### Why two sources & timezone handling

Demand is largely a function of weather, so joining government load data with an
independent weather feed lets the model learn the temperature → demand relationship.
EIA timestamps are **UTC**; everything is standardized to UTC on ingestion, then
localized to **America/Chicago** for the calendar features.

---

## Stage B — SQL Join + Calendar Features (DuckDB)

The demand and weather tables are joined with **SQL (DuckDB)** on the UTC timestamp,
producing one row per hour with both demand and temperature aligned. Calendar features
(hour, day of week, month, weekend flag) are derived from the local timestamp. See
[`sql/join_demand_weather.sql`](../sql/join_demand_weather.sql).

## Stage C — Cleaning + Data-Quality Report

Detect and handle missing hours, duplicate timestamps, and outliers, and produce a
data-quality report documenting completeness. A documented DQ check is a deliberate
rigor signal — a forecast is only trustworthy if the data underneath it is verified.

---

## Stage D — Exploratory Data Analysis

Confirm the patterns the model needs to learn actually exist: daily/weekly seasonality,
the U-shaped temperature-demand relationship (demand rises in both heat and cold), and
overall trend/outliers. EDA is what justifies the feature choices in the next stage.

## Stage E — Feature Engineering

All features are **leakage-safe** — they use only information available at prediction
time, never the future.

- **Lagged demand**: 24h, 48h, 168h (one week ago).
- **Rolling statistics**: rolling mean and std, each `shift(1)` so the current hour
  never predicts itself.
- **Degree-days**: heating and cooling degree-days around an 18 °C comfort base.
- **Cyclical encodings**: sin/cos of hour-of-day and day-of-year.

---

## Stage F — Modeling: Baseline vs. Gradient Boosting

- **F1 — Seasonal-naive baseline**: "this hour looks like the same hour last week."
- **F2 — Gradient-boosted model**: learns the nonlinear interactions between lags,
  weather, and calendar features.

The baseline is the honest yardstick — a model that can't beat "last week" adds no value.

## Stage G — Walk-Forward Backtest + Calibrated Intervals

- **Walk-forward evaluation**: train on the past, predict the immediately-following
  period, roll forward, repeat. Random splits leak future into past and overstate
  accuracy — this is the time-series-correct method.
  **Result: 2.67% MAPE — 69% better than the seasonal-naive baseline (8.70%).**
- **Conformalized Quantile Regression (CQR)**: quantile models predict the 10th/50th/90th
  percentiles; a held-out calibration set computes a conformal correction so the intervals
  hit their stated coverage.
  **Result: 82.5% empirical coverage at an 80% target.**

---

## Stage H — The Grid Planner App

A deployed Streamlit app: pick any day, adjust a temperature scenario, and see the
forecast band, predicted peak demand and peak hour, a worst-case figure, and a
plain-English operator recommendation. See [`app/main.py`](../app/main.py).

**Live app:** https://ercot-grid-planner1.hub.zerve.cloud
