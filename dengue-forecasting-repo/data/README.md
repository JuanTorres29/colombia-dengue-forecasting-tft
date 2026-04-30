# Data folder

This folder contains the panel data consumed by the pipeline.

## Files

- **`final_dataset_sample.csv`** (committed, ~4.4 MB):
  A stratified sample of 50 municipalities drawn from the full 376-municipality
  panel. Municipalities were sampled uniformly within quartiles of historical
  total cases, so the sample preserves the mix of high-, medium- and
  low-endemicity municipalities. Use this to verify that the pipeline runs
  end-to-end without distributing the full dataset.

- **`final_dataset.csv`** (NOT committed, ~33 MB):
  Full panel used in the thesis: 376 municipalities × 782 weeks (2010–2024),
  294,032 rows. To use it, place the file here and either:
  - update `DATA_PATH` at the top of the notebook to point to it, or
  - simply delete the sample and rename the full file to
    `final_dataset_sample.csv`.

## Schema

| Column | Type | Description |
|---|---|---|
| `COD_MUN_N` | int | DIVIPOLA municipality code |
| `ANO` | int | Year |
| `SEMANA` | int | Epidemiological week within year (1–52/53) |
| `week_start` | str | ISO date of the start of the week (YYYY-MM-DD) |
| `time_idx` | int | Global integer week index, shared across municipalities |
| `MES` | int | Calendar month |
| `casos_totales` | int | Total weekly dengue cases (target variable) |
| `casos_0_4` | int | Weekly cases, age 0–4 |
| `casos_5_14` | int | Weekly cases, age 5–14 |
| `casos_15_64` | int | Weekly cases, age 15–64 |
| `casos_65_plus` | int | Weekly cases, age 65+ |
| `casos_m` | int | Weekly cases, male |
| `casos_f` | int | Weekly cases, female |
| `temp_mean` | float | Weekly mean temperature (not used in the final model) |
| `temp_mean_missing` | int | Indicator flag for `temp_mean` imputation |
| `prec_total` | float | Weekly accumulated precipitation (mm) |
| `ndvi_mean` | float | Monthly NDVI averaged spatially over the municipality |
| `oni_anom` | float | Oceanic Niño Index anomaly (rolling 3-month) |
| `oni_total` | float | Total ONI sea-surface temperature value |
| `Flujo_in` | float | Gravitational inbound connectivity flow |
| `poblacion` | int | Municipal population (Censo 2018) |

## How to regenerate the full dataset

The full dataset is built outside this repository, by combining:

1. SIVIGILA case-level records aggregated by municipality, week, and age group
2. IDEAM daily precipitation aggregated to weekly totals per municipality
3. MODIS MOD13Q1 NDVI tiles spatially averaged over each municipal polygon and
   matched to weekly observations by calendar month
4. NOAA ONI monthly index broadcast to weeks within the central month of each
   trimester
5. Gravitational flow computed from DANE 2018 population and centroid distances
   from the IDEAM municipal shapefile (see thesis section 6.2.4 for the formula)
6. Municipality filtering: only municipalities with ≤20% missing precipitation
   AND ≥200 total cases AND ≥26 weeks with at least one case are retained

A complete preprocessing script is outside the scope of this repository but
the methodology section of the accompanying thesis describes each step.
