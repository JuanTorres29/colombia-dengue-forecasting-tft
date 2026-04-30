# Dengue Forecasting in Colombia

Reproducible code for forecasting weekly municipality-level dengue cases in
Colombia using a **Temporal Fusion Transformer (TFT)** and three baseline
models (Naive, ARIMA, LSTM). Companion repository for the undergraduate
thesis "Predicción de casos de dengue en Colombia mediante Temporal Fusion
Transformer (TFT)" (Pontificia Universidad Javeriana, 2026).

## What this repository contains

```
.
├── notebooks/
│   └── dengue_forecasting.ipynb   # main pipeline (train + tune + test)
├── data/
│   └── final_dataset_sample.csv   # 50-municipality stratified sample (4.4 MB)
├── results/                        # created at runtime; gitignored
├── requirements.txt
├── .gitignore
└── README.md
```

The main notebook has two cells:

1. **Pipeline cell** — full training and evaluation pipeline:
   - light cross-validation on TFT (validation folds 2021, 2022, 2023)
   - two-stage hyperparameter tuning on the 2023 fold
     - Stage 1: architecture search (`hidden_size × attention_head_size`)
     - Stage 2: loss-weight search (`w_zero × w_pos × w_outbreak`)
   - final test on 2024 with all four models (TFT, NAIVE, ARIMA, LSTM)
2. **Anticipation fix cell** — post-hoc correction of the lead-time metric
   in the comparison Excel.

## Dataset

The full dataset (`final_dataset.csv`) is a panel of **376 Colombian
municipalities × 782 epidemiological weeks** (2010–2024, 294,032 rows) built
from public sources:

| Variable | Source |
|---|---|
| Weekly dengue cases by municipality and age group (0-4, 5-14, 15-64, 65+) | SIVIGILA / Instituto Nacional de Salud |
| Weekly accumulated precipitation | IDEAM (DHIME portal) |
| Monthly NDVI (MOD13Q1) averaged by municipality | NASA MODIS via Google Earth Engine |
| Oceanic Niño Index (ONI) | NOAA Climate Prediction Center |
| Municipal centroids and shapefile | IDEAM |
| Municipal population (Censo 2018) | DANE |
| Gravitational connectivity flow (`Flujo_in`) | Computed from population + centroids |

The full dataset is publicly available on Kaggle:

👉 https://www.kaggle.com/datasets/juantorr/dengue-and-climate-data-colombia-20102024

A **small stratified sample of 50 municipalities** is included in this
repository under `data/final_dataset_sample.csv` to ensure the pipeline can
be executed quickly end-to-end.

To run the full experiment, download the dataset from Kaggle and replace
`data/final_dataset_sample.csv` with `data/final_dataset.csv`, then update
`DATA_PATH` at the top of the notebook.

### Required columns of the input CSV

```
COD_MUN_N, ANO, SEMANA, week_start, time_idx,
casos_totales, casos_0_4, casos_5_14, casos_15_64, casos_65_plus,
prec_total, ndvi_mean, oni_anom, Flujo_in, poblacion
```

`time_idx` must be a monotonically increasing integer week index that
matches across municipalities (week 0 is the same calendar week for all
municipalities).

## Setup

Python 3.10+ is recommended.

```bash
# Create a virtual environment
python -m venv venv
source venv/bin/activate          # Linux / macOS
# venv\Scripts\activate            # Windows

# Install dependencies
pip install -r requirements.txt
```

GPU is strongly recommended for the TFT training. The pipeline detects
CUDA automatically; on CPU the run is feasible but slow on the full panel.

## Running the pipeline

```bash
cd notebooks
jupyter notebook dengue_forecasting.ipynb
```

Then execute the two cells in order. By default, results land under
`../results/` relative to the notebook, with the following structure:

```
results/
├── cv/
│   ├── fold_val_2021/<model>/...
│   ├── fold_val_2022/<model>/...
│   ├── fold_val_2023/<model>/...
│   ├── _tuning_tft_2023/
│   │   ├── arch_*/...           # Stage-1 architecture trials
│   │   ├── weights_*/...        # Stage-2 weight trials
│   │   ├── best_architecture.json
│   │   ├── best_config.json
│   │   └── tuning_results.csv
│   └── cv_summary_report.xlsx
└── final_test/
    └── final_test_2024/
        ├── tft/   naive/   arima/   lstm/
        ├── final_test_2024_comparison_report.xlsx
        └── final_test_2024_comparison_report_CORREGIDO.xlsx
```

The pipeline supports **resume**: if it is interrupted, re-running the
cell skips trials that already finished (their results are persisted as
JSON inside their trial folders).

## Reproducibility notes

- A single global random seed (`SEED = 42`) is set at the top of the
  pipeline. Lightning's `seed_everything(SEED, workers=True)` is also
  invoked.
- The 2024 test year is held out from every step of CV and tuning.
- Hyperparameter search is driven by a composite score documented in the
  `_score_tft_trial` function; the final selection is persisted in
  `best_config.json`.

## Citation

If you use this code, please cite the thesis:

> Torres Contreras, J. A. (2026). *Predicción de casos de dengue en
> Colombia mediante Temporal Fusion Transformer (TFT)*. Undergraduate
> thesis, Pontificia Universidad Javeriana, Departamento de Matemáticas,
> Programa Ciencia de Datos.

## License

This code is released for academic and research purposes. Underlying
SIVIGILA, IDEAM, MODIS, NOAA, and DANE data are subject to the terms of
their respective providers.
