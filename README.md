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
│   ├── dengue_forecasting.ipynb           # main pipeline (CV + tuning + test) + interpretability
│   └── reproduce_thesis_results.ipynb     # loads checkpoints and reproduces test-2024 metrics
├── checkpoints/
│   ├── tft_final_test_2024.ckpt           # TFT trained for the thesis
│   └── lstm_final_test_2024.ckpt          # LSTM trained for the thesis
├── data/
│   └── final_dataset_sample.csv           # 50-municipality stratified sample (4.4 MB)
├── results/                                # created at runtime; gitignored
├── requirements.txt
├── .gitignore
└── README.md
```

There are two notebooks with different purposes:

1. **`dengue_forecasting.ipynb`** — full training pipeline. Runs the CV,
   loss-weight tuning, and final test from scratch on the dataset of your
   choice (sample by default). Use it to retrain or to experiment with
   variations of the pipeline.
2. **`reproduce_thesis_results.ipynb`** — standalone reproduction notebook.
   Loads the TFT and LSTM checkpoints stored in `checkpoints/`, reruns
   NAIVE and ARIMA from scratch (they are deterministic and need no
   weights), and produces the metrics reported in the thesis. Use it to
   reproduce the published numbers without retraining.

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
| Training epochs (final test)                      | median of CV-light optimal epochs                  |
| Learning rate                                     | 0.003                                              |
| Gradient clipping                                 | 0.1                                                |
| `hidden_size`                                     | 64                                                 |
| `attention_head_size`                             | 4                                                  |
| `dropout`                                         | 0.15                                               |
| `hidden_continuous_size`                          | 32                                                 |
| Loss                                              | Negative Binomial Distribution Loss                |
| Target normalization                              | `GroupNormalizer` per municipality with `log1p`    |

Both the CV-light step (which selects the median number of training
epochs) and the final test refit run with this same architecture. The
loss-weight tuning step also keeps it fixed and only searches over
`(w_zero, w_positive, w_outbreak)`.

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

A **stratified sample of 50 municipalities** is included in this
repository under `data/final_dataset_sample.csv` so the pipeline can be
executed end-to-end after a fresh clone.

> **Important:** the sample is for verifying that the code runs end-to-end.
> The thesis numbers were produced on the full 376-municipality panel and
> the checkpoints in `checkpoints/` were trained on it; reproducing the
> thesis metrics requires the full dataset because the TFT's municipality
> embedding has 376 entries and is not compatible with a smaller set.

To use the full dataset:

1. Download `final_dataset.csv` from the Kaggle link above.
2. Place it under `data/`.
3. The reproduction notebook (`reproduce_thesis_results.ipynb`) will pick
   it up automatically. For the training notebook
   (`dengue_forecasting.ipynb`), change the `DATA_PATH` line at the top of
   the pipeline cell from `final_dataset_sample.csv` to
   `final_dataset.csv`.

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
python -m venv venv
source venv/bin/activate          # Linux / macOS
# venv\Scripts\activate            # Windows

pip install -r requirements.txt
```

GPU is recommended for training the TFT from scratch. CPU works but is
considerably slower on the full panel. The reproduction notebook runs
fine on CPU because it only does inference.

## How to use this repository

### Reproduce the thesis numbers (no training)

```bash
cd notebooks
jupyter notebook reproduce_thesis_results.ipynb
```

Run the cells in order. The notebook reads the dataset (full or sample),
loads `checkpoints/tft_final_test_2024.ckpt` and
`checkpoints/lstm_final_test_2024.ckpt`, and rebuilds the test-set
predictions and metrics for all four models. With the full dataset and
the included checkpoints the numbers should match the thesis.

### Train from scratch

```bash
cd notebooks
jupyter notebook dengue_forecasting.ipynb
```

Run the pipeline cell first, then the interpretability cell once it has
finished. Default outputs land under `../results/` relative to the
notebook:

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

## Reproducibility

There are two senses of "reproducible" worth distinguishing:

**Reproducing the thesis numbers.** Use
`reproduce_thesis_results.ipynb` with the full dataset. Inference from a
fixed checkpoint is deterministic, so the test-2024 metrics will match
the thesis.

**Reproducibility of retraining.** The training pipeline applies a
single global seed (`SEED = 42`), `seed_everything(workers=True)`,
explicit `torch.Generator` for the LSTM DataLoader, `deterministic=True`
on every Lightning `Trainer`, cuDNN configured for determinism, and a
per-(fold, model) reseeding step before each `trainer.fit`. Together
these make a from-scratch CPU run deterministic across executions.

A few caveats worth knowing about:

- On GPU, even with all the deterministic flags set, small residual
  differences are possible because some kernels in CUDA / cuDNN do not
  have deterministic implementations. These differences are normally
  under `1e-5` per weight and do not change the qualitative conclusions.
- The bundled checkpoints were trained with an earlier version of the
  pipeline that did not have the per-fold reseeding patch. A
  from-scratch retraining with the current code therefore gives a
  similar but not identical model. If you need the exact thesis numbers,
  use the reproduction notebook.

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
