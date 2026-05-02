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
│   └── dengue_forecasting.ipynb   # main pipeline (CV + tuning + test) + interpretability
├── data/
│   └── final_dataset_sample.csv   # 50-municipality stratified sample (4.4 MB)
├── results/                        # created at runtime; gitignored
├── requirements.txt
├── .gitignore
└── README.md
```

The main notebook has two cells:

1. **Pipeline cell** — full training and evaluation pipeline:
   - **Light cross-validation** (folds 2021, 2022, 2023) trains the TFT on
     each validation year and records the optimal training epoch (lowest
     `val_loss`) per fold. The median across folds is the number of epochs
     used for the final test refit.
   - **Loss-weight tuning** (single stage) over the 2023 fold, with the
     final TFT architecture fixed (see table below).
   - **Final test on 2024** with all four models (TFT, NAIVE, ARIMA, LSTM),
     using the median epochs from CV and the best loss weights from tuning.
2. **Interpretability cell** — variable importance and attention analysis
   on the trained TFT (variable selection networks, temporal attention,
   per-regime breakdowns, Excel workbook with sheets A–K).

## TFT specification

The TFT architecture is **fixed** to the specification reported in the
thesis. The pipeline does **not** search over architecture; only loss
weights are tuned.

| Hyperparameter                                    | Value                                              |
|---------------------------------------------------|----------------------------------------------------|
| Max encoder length                                | 26 weeks                                           |
| Min encoder length                                | 13 weeks                                           |
| Max prediction horizon                            | 4 weeks                                            |
| Batch size                                        | 64                                                 |
| Training epochs (final test)                      | 5 (median selected via CV)                         |
| Learning rate                                     | 0.003                                              |
| Gradient clipping                                 | 0.1                                                |
| `hidden_size`                                     | 64                                                 |
| `attention_head_size`                             | 4                                                  |
| `dropout`                                         | 0.15                                               |
| `hidden_continuous_size`                          | 32                                                 |
| Loss                                              | Negative Binomial Distribution Loss                |
| Target normalization                              | `GroupNormalizer` per municipality with `log1p`    |

The CV step that selects the median number of training epochs intentionally
runs with `attention_head_size = 8`. The final-test refit then uses the
fixed final architecture (`attention_head_size = 4`) together with the
median epochs from that CV. This separation is documented in the thesis
methodology.

## Dataset

The full dataset (`final_dataset.csv`) is a panel of **376 Colombian
municipalities × 782 epidemiological weeks** (2010–2024, 294,032 rows)
built from public sources:

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

A **stratified sample of 50 municipalities** is included in this repository
under `data/final_dataset_sample.csv` so the pipeline can be executed
end-to-end after a fresh clone.

The notebook uses the sample by default. To reproduce the full experiment,
download `final_dataset.csv` from the Kaggle link above, place it under
`data/`, and change the `DATA_PATH` line at the top of the pipeline cell
from `final_dataset_sample.csv` to `final_dataset.csv`.

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
├── cv/
│   ├── fold_val_2021/tft/...           # CV-light: only val_loss + selected epoch
│   ├── fold_val_2022/tft/...
│   ├── fold_val_2023/tft/...
│   └── _tuning_tft_2023/
│       ├── weights_*/...               # loss-weight trials
│       ├── best_config.json
│       └── tuning_results.csv
├── final_test/
│   └── final_test_2024/
│       ├── tft/   naive/   arima/   lstm/
│       ├── final_test_2024_comparison_report.xlsx
│       └── tft/interp_artifacts/       # inputs for the interpretability cell
└── interp_manifest_pointer.json
```

The pipeline supports **resume**: if it is interrupted, re-running the
cell skips folds and trials that already finished (their results are
persisted as JSON inside their respective folders).

## Reproducibility notes

- A single global random seed (`SEED = 42`) is set at the top of the
  pipeline. Lightning's `seed_everything(SEED, workers=True)` is also
  invoked.
- The 2024 test year is held out from every step of CV and tuning.
- The TFT architecture is fixed; the search budget is concentrated on the
  loss-weight grid. The selection criterion is the composite score
  documented in `_score_tft_trial`. The final selection is persisted in
  `cv/_tuning_tft_2023/best_config.json`.
- The number of epochs for the final TFT refit is the median of the
  per-fold optimal epochs recorded during the CV-light step.

## Citation

If you use this code, please cite the thesis:

> Torres Contreras, J. A. (2026). *Predicción de casos de dengue en
> Colombia mediante Temporal Fusion Transformer (TFT)*. Undergraduate
> thesis, Pontificia Universidad Javeriana, Programa Ciencia de Datos.

## License

This code is released for academic and research purposes. Underlying
SIVIGILA, IDEAM, MODIS, NOAA, and DANE data are subject to the terms of
their respective providers.
