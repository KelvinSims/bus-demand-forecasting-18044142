# Predicting Passenger Demand to Support Bus Timetable Optimisation

**MSc Data Science — CSCT Masters Project (UFCF9Y-60-M)**
**Student number: 18044142**

Artefact accompanying the project report. The notebook implements a reproducible pipeline that forecasts short-term bus passenger demand from open MTA data, converts those forecasts into service-frequency recommendations, and compares the recommendations against the timetable New York's MTA actually operates.

---

## What the pipeline does

1. **Data engineering** — loads the raw MTA hourly ridership export (36.4 million rows), aggregates to route–hour grain, persists to Parquet.
2. **Feature engineering** — leakage-safe lag, rolling and slot-based features computed within each route.
3. **Forecasting** — three transparent baselines plus a benchmarked LightGBM model, evaluated on chronological splits.
4. **Frequency recommendation** — a Ceder-style rule converting predicted demand into buses per hour and headways.
5. **Comparison** — scheduled frequencies extracted from MTA GTFS feeds and compared against the recommendations.

## Headline results

| Model | MAE | RMSE |
|---|---:|---:|
| Naïve: same hour yesterday | 26.28 | 67.16 |
| Naïve: same hour last week | 16.54 | 41.06 |
| Historical average | 18.99 | 44.61 |
| LightGBM v2 (final) | **10.46** | **21.90** |

A 37% reduction in mean absolute error against the strongest baseline. In the frequency comparison, recommendations closely tracked the operated timetable on Q58 (mean signed gap +0.6 buses/hour) but systematically under-recommended on B100 (−6.2), a divergence traced to the route-total load proxy that route-level open data forces.

---

## Repository contents

```
01_data_exploration.ipynb          Full pipeline, documented cell by cell
requirements.txt                   Python packages needed
figures/                           Figures used in the report
results/                           Model and comparison output tables
```

Raw and intermediate data files are **not** included (see below).

---

## Data: how to obtain it

The raw datasets are too large for version control and are freely available from the original sources.

**1. MTA hourly bus ridership (demand data)**

- Source: https://data.ny.gov/d/kv7t-n8in
- Filter to `transit_timestamp` between 2023-01-01 and 2024-12-31, then export as CSV.
- Save as `MTA_Bus_Hourly_Ridership_2023-24.csv` in the project root.
- Accessed 22 July 2026.

**2. MTA GTFS timetable feeds (supply data)**

- NYCT bus feed (for Q58): `http://web.mta.info/developers/data/nyct/bus/google_transit_queens.zip`
- MTA Bus Company feed (for B100): `http://web.mta.info/developers/data/busco/google_transit.zip`
- Unzip into `data/gtfs_q/` and `data/gtfs_busco/` respectively.
- Accessed 7th June 2026.
- Note: MTA's per-borough feeds are full-network route catalogues; trips are partitioned by operating division, so B100 (an MTA Bus Company route) is scheduled in the Bus Company feed rather than the Brooklyn feed. SIM express schedules are published in a separate feed not used here.

Feeds are updated quarterly, so a later download will reflect a different schedule than the one analysed.

---

## Running the notebook

```bash
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

**macOS only:** LightGBM requires OpenMP, which is not installable via pip:

```bash
brew install libomp
```

Then open `01_data_exploration.ipynb`, select the venv as the kernel, and run all cells from the top. Expect roughly ten minutes: the CSV parse and two model fits dominate.

All configuration (file paths, split dates, random seed, capacity and policy parameters) is in the first cell.

---

## Reproducibility notes

- A fixed random seed and pinned chronological split dates make every reported figure reproducible.
- Expensive intermediates (aggregated data, engineered features, trained models) are written to disk, so sessions after the first resume in seconds.
- The pipeline is verified by a clean restart-and-run-all execution.
