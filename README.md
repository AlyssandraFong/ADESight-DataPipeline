# ADESight — ML Data Pipeline

Machine learning pipeline for predicting adverse drug reactions (ADRs) from FAERS post-marketing surveillance data, as part of the ADESight pharmacovigilance analytics platform.

## Overview

This pipeline frames ADR prediction as a **multi-label classification problem**: given patient/report-level features, predict the likelihood of **10 ADR labels** across **10 tracked drugs**. Five model families are trained and compared, with the best-performing architecture selected for deployment in the ADESight web application.

## Data

- **Source:** FDA Adverse Event Reporting System (FAERS), 2025 release
- **Processed format:** Parquet, ~122,923 rows
- **Cleaning:** individual source tables cleaned, then merged, then cleaned again — all row-level operations, safe to perform before the split

> **Note:** raw and processed FAERS data are **not included in this repository** due to file size constraints. Only the Jupyter notebooks are provided. To reproduce the pipeline, download FAERS 2025 data from the [FDA FAERS public dashboard](https://www.fda.gov/drugs/questions-and-answers-fdas-adverse-event-reporting-system-faers/fda-adverse-event-reporting-system-faers-public-dashboard) and run the preprocessing notebook to regenerate the parquet file locally.

## Repository Contents

This repo contains the pipeline notebooks only — no data or trained model artifacts are checked in.

| Notebook | Purpose |
|---|---|
| *Notebook 1-5* | Data preprocessing & EDA |
| *Notebook 6* | Model training & BayesSearchCV tuning |
| *Notebook 7* | OOF threshold calibration & evaluation |

## Pipeline Stages

The pipeline follows a fixed order to guard against data leakage:

```
Clean individual tables → Merge tables → Clean merged data → Split
  → Feature engineering → Impute → Feature importance → Encode → Scale → Train
```

Table cleaning (both pre- and post-merge) is row-level only — dropping malformed rows, fixing types, deduping — so it's safe to do before the split. Everything from imputation onward is fit on the training split only and then applied to validation/test.

Key leakage vectors identified and closed during development:
- Imputation performed pre-split
- Feature importance computed on full dataset instead of train-only
- `eval_set` early stopping leaking into OOF threshold derivation
- Country frequency encoding computed before the split

## Models

Five model families, with XGBoost implemented in two architectural variants:

| Model | Notes |
|---|---|
| Logistic Regression | `class_weight="balanced"` |
| Decision Tree | `class_weight="balanced"` |
| Random Forest | `class_weight="balanced"` |
| LightGBM | `class_weight="balanced"` |
| XGBoost — MultiOutput | shared hyperparameters across all labels, `scale_pos_weight` |
| XGBoost — Loop | per-label `scale_pos_weight`, shared BayesSearchCV config |

**Hyperparameter tuning:** `BayesSearchCV` (scikit-optimize)
**Cross-validation:** `MultilabelStratifiedKFold` (skmultilearn / iterstrat)
**Class imbalance handling:** `class_weight="balanced"` for Logistic Regression, Decision Tree, Random Forest, and LightGBM; `scale_pos_weight` for both XGBoost variants. SMOTE was evaluated and rejected — leakage risk and clinical implausibility of synthetic FAERS samples.
**Threshold calibration:** F2-optimized thresholds derived from out-of-fold (OOF) predictions, not validation-set labels directly — this keeps the validation set legitimately reusable while the test set is touched only once.

## Final Model: XGBoost Loop

XGBoost Loop was selected as the final model. Full test-set metrics are reported in the thesis (Chapter 5) rather than here.

**Why Loop over MultiOutput:** the MultiOutput variant's baseline was never beaten by tuning — a structural consequence of sharing hyperparameters across labels with widely varying imbalance ratios (2.7x–28.2x between drug/reaction pairs). The Loop variant varies `scale_pos_weight` per label while keeping the BayesSearchCV configuration shared (not independent tuning per label — this distinction is documented explicitly for academic defensibility).

**Known limitation:** `reaction_DEATH` (3.4% prevalence) shows an AP/ROC-AUC divergence, characteristic of extreme class imbalance. This is documented as a data limitation rather than a modelling flaw.

**Feature note:** `reporter_type` (`occp_cod`) was deliberately excluded as an input feature — it reflects who filed the report, not the patient's clinical state, and would bias predictions toward submitter identity rather than clinical signal.

## Benchmarking

Performance is compared against six published studies; see Chapter 5 for the full comparison. The gap on AP relative to some benchmarks is attributed to feature scope constraints — FAERS demographic features only, versus biological/molecular features used in competing studies.

## Tech Stack

- Python (Kaggle, GPU/CUDA)
- `xgboost`, `lightgbm`, `scikit-learn`
- `scikit-optimize` (BayesSearchCV)
- `skmultilearn` (MultilabelStratifiedKFold / IterativeStratification)
- `pandas`, `numpy`, `joblib`

## Outputs

Running the notebooks end-to-end produces (not included in this repo):

- Trained model artifacts (`.pkl`)
- Per-model OOF-calibrated thresholds (`thresholds_*`)
- JSON configs for drug/reaction column ordering (required for consistent inference in the web app)

These artifacts are what the ADESight web app's backend loads at runtime — regenerate them locally by running the notebooks before starting the app.

## Repository Conventions

- Full script/notebook rewrites are preferred over partial patches
- Naming conventions: `search_rf3`, `best_xgb_loop_params`, `model_xgb_loop_tuned`, `thresholds_*`

---
*Part of the ADESight Final Year Project. See the main [ADESight Website Platform](https://github.com/AlyssandraFong/ADESight-Platform.git) for the full-stack application.*
