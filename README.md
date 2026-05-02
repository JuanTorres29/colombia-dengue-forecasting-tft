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
   - two-stage hyperparameter tuning on the 2023 validation fold
     - Stage 1: architecture search (`hidden_size × attention_head_size`)
     - Stage 2: loss-weight search (`w_zero × w_pos × w_outbreak`)
   - light cross-validation (folds 2021, 2022, 2023) using the selected
     configuration to choose the median number of training epochs
   - final test on 2024 with all four models (TFT, NAIVE, ARIMA, LSTM)
2. **Interpretability cell** — variable importance and attention analysis on
   the trained TFT (variable selection networks, temporal attention,
   per-regime breakdowns, Excel workbook with sheets A–K).

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

The notebook is configured to use the sample by default, so it runs
end-to-end after a fresh clone. To reproduce the full experiment, download
`final_dataset.csv` from the Kaggle link above, place it under `data/`, and
change the `DATA_PATH` line at the top of the pipeline cell from
`final_dataset_sample.csv` to `final_dataset.csv`.

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

Run the cells in order: the pipeline cell first, and the interpretability
cell once it has finished. By default, results land under `../results/`
relative to the notebook, with the following structure:

```
results/
├── tuning/
│   └── _tuning_tft_2023/
│       ├── arch_*/...                # Stage-1 architecture trials
│       ├── weights_*/...             # Stage-2 weight trials
│       ├── best_architecture.json
│       ├── best_config.json
│       └── tuning_results.csv
├── cv/
│   ├── fold_val_2021/tft/...
│   ├── fold_val_2022/tft/...
│   ├── fold_val_2023/tft/...
│   └── _CV_SUMMARY.json              # median selected epochs
├── final_test/
│   └── final_test_2024/
│       ├── tft/   naive/   arima/   lstm/
│       ├── final_test_2024_comparison_report.xlsx
│       └── tft/interp_artifacts/     # inputs for the interpretability cell
└── interp_manifest_pointer.json
```

The pipeline supports **resume**: if it is interrupted, re-running the
cell skips trials and folds that already finished (their results are
persisted as JSON inside their respective folders).

## Reproducibility notes

- A single global random seed (`SEED = 42`) is set at the top of the
  pipeline. Lightning's `seed_everything(SEED, workers=True)` is also
  invoked.
- The 2024 test year is held out from every step of tuning and CV.
- Hyperparameter search is driven by a composite score documented in the
  `_score_tft_trial` function; the final selection is persisted in
  `best_config.json`.

## Citation

If you use this code, please cite the thesis:

> Torres Contreras, J. A. (2026). *Predicción de casos de dengue en
> Colombia mediante Temporal Fusion Transformer (TFT)*. Undergraduate
> thesis, Pontificia Universidad Javeriana, Programa Ciencia de Datos.

## License

This code is released for academic and research purposes. Underlying
SIVIGILA, IDEAM, MODIS, NOAA, and DANE data are subject to the terms of
their respective providers.

## License

This code is released for academic and research purposes. Underlying
SIVIGILA, IDEAM, MODIS, NOAA, and DANE data are subject to the terms of
their respective providers.
