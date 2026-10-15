# SEER Lung Cancer Survival Analysis
### Machine Learning Model Documentation

> **Data:** SEER*Stat Version 9.0.43.0 &nbsp;•&nbsp; Extracted May 10, 2026 &nbsp;•&nbsp; 17 Registries, 2000–2022
>
> **Last updated:** October 2026 (v4: target-leakage fix; see [Changelog](#9-changelog))

---

## Summary

Three models predicting lung cancer survival for ~865,000 SEER patients diagnosed 2000–2022:

| Model | Test result | Takeaway |
|---|---|---|
| Random Survival Forest | C-index 0.72 | Best for ranking patient risk; handles censoring |
| Decision Tree Regressor | R² 0.47, MAE 15.9 months | Interpretable baseline |
| PyTorch Neural Network | R² −0.32 | Underperforms the trees; diagnosis in [Section 4.6](#46-performance-results-v4) |

**Key finding:** earlier (v3) results were inflated by target leakage, because cause of death was used as a feature. Removing it dropped Decision Tree R² from 0.75 to 0.47 and RSF C-index from 0.76 to 0.72. The v4 numbers above are the corrected results. Surgery, stage, and age are the strongest predictors in both tree models.

---

## Table of Contents
0. [Summary](#summary)
1. [Data Source](#1-data-source)
2. [Shared Preprocessing](#2-shared-preprocessing-steps)
3. [Model 1: Decision Tree Regressor](#3-model-1-decision-tree-regressor)
4. [Model 2: PyTorch Neural Network](#4-model-2-pytorch-neural-network)
5. [Model 3: Random Survival Forest](#5-model-3-random-survival-forest)
6. [Model Comparison](#6-model-comparison)
7. [Environment Setup](#7-environment-setup)
8. [Known Limitations / To-Do](#8-known-limitations--to-do)
9. [Changelog](#9-changelog)

---

## 1. Data Source

This analysis uses data extracted from the Surveillance, Epidemiology, and End Results (SEER) Program, maintained by the National Cancer Institute. The dataset covers malignant lung and bronchus cancer cases across 17 registries from 2000 to 2022.

> **Data access:** SEER data is not included in this repository, per the SEER Research Data Use Agreement. To reproduce this analysis, request SEER Research Data access at [seer.cancer.gov/data/access.html](https://seer.cancer.gov/data/access.html) and build an extract using the variable list in [Section 1.3](#13-columns-in-the-extract).
>
> **Not included in this repository:**
> - SEER data and patient-level outputs (prediction CSVs)
> - Trained model files (`.pkl`, `.pth`). The Random Survival Forest model is ~340 MB, over GitHub's file size limit. Running the scripts in [Section 7](#7-environment-setup) regenerates all models and outputs.

### 1.1 Citation

Surveillance, Epidemiology, and End Results (SEER) Program (www.seer.cancer.gov) SEER\*Stat Database: Incidence – SEER Research Data, 17 Registries, Nov 2024 Sub (2000–2022) – Linked To County Attributes – Time Dependent (1990–2023) Income/Rurality, 1969–2023 Counties, National Cancer Institute, DCCPS, Surveillance Research Program, released April 2025, based on the November 2024 submission.

### 1.2 Dataset Overview

| Property | Value |
|---|---|
| Selection criteria | Malignant behavior; known age; Site recode ICD-O-3 2023 Revision Expanded = Lung and Bronchus |
| Total records extracted | 1,105,791 |
| Records after exclusions | 887,217 |
| Records with valid survival data | 865,036 |
| Records used in RSF (after removing 0-month survival) | 767,537 |
| Diagnosis years | 2000–2022 |
| Registries | 17 (SEER Research Data) |

> ⚠️ **Cohort limitation:** the current extraction contains only two cause-of-death values, `Lung And Bronchus` (death from lung cancer) and `Alive`. Patients who died of other causes were excluded upstream. See [Section 8](#8-known-limitations--to-do).

### 1.3 Columns in the Extract

| Column Name | Type | Role |
|---|---|---|
| Patient ID | Identifier | Unique patient identifier (not a model feature) |
| Age recode with <1 year olds and 90+ | Categorical → Numeric | Feature; converted to midpoint (`Age_Mean`) |
| Sex | Categorical | Feature (2 unique values) |
| Marital status at diagnosis | Categorical | Feature (7 unique values) |
| Year of diagnosis | Numeric | Feature |
| SEER historic stage A (1973–2015) | Categorical | Feature (5 unique values) |
| Derived Summary Grade 2018 (2018+) with NULL | Categorical | Feature (11 unique values; ~77% missing, only recorded 2018+) |
| RX Summ–Surg Prim Site (1998+) | Categorical | Feature (25 unique values) |
| Time from diagnosis to treatment in days recode with NULL | Categorical → Numeric | Feature; ~35% missing |
| Tumor Size Over Time Recode (1988+) with NULL | Categorical → Numeric | Feature; ~28% missing |
| Survival months with NULL | Numeric | **Target** (survival time) |
| COD to site recode ICD-O-3 2023 Revision Expanded (1999+) | Categorical | **Outcome definition only**: RSF event indicator. **Excluded as a feature in all models** (target leakage; cause of death is only known after the outcome) |

---

## 2. Shared Preprocessing Steps

All three models share the following preprocessing pipeline before model-specific steps diverge.

| Step | Operation | Notes |
|---|---|---|
| Load | `pd.read_csv(..., low_memory=False)` | Avoids mixed-type `DtypeWarning` on the grade column |
| Deduplication | `df.drop_duplicates()` | Removes 235 duplicate rows |
| Target filtering | Drop rows where survival months is null | Retains 865,036 patients |
| Age engineering | Extract midpoint from age range strings | E.g. `"65-69 years"` → 67.0; `"90+"` → 92 |
| Tumor size | Extract numeric mm from string field | NULL/Unknown → NaN |
| Treatment days | Extract integer from string field | NULL/Unknown → NaN |
| Categorical encoding | `LabelEncoder` per column; NaN → `'Unknown'` | Cause of death is **not** encoded as a feature |
| Missing imputation | Median fill for `Age_Mean`, `Tumor_Size_mm`, `Treatment_Days` | Decision Tree uses native missing-value handling instead |
| Standardization | `StandardScaler` (mean=0, std=1) on numeric features | Applied after imputation |

---

## 3. Model 1: Decision Tree Regressor

**Script:** `decision_tree_regressor.py`

### 3.1 Purpose

Predicts survival months as a continuous regression target. Serves as the interpretable baseline and as the source of the feature importances used to weight Neural Network inputs.

### 3.2 Key Differences from Other Models

- **Target:** raw survival months (regression; no log transform, no censoring)
- **Missing values** handled natively by scikit-learn ≥ 1.3, so no imputation for `Tumor_Size_mm` or `Treatment_Days`
- Numeric feature columns use a `_Scaled` suffix (`Age_Mean_Scaled`, `Tumor_Size_mm_Scaled`, `Treatment_Days_Scaled`)
- **9 features** (cause of death removed)

### 3.3 Model Parameters

| Parameter | Value | Rationale |
|---|---|---|
| `max_depth` | 10 | Limits tree depth to prevent overfitting |
| `min_samples_split` | 100 | Requires 100 samples before splitting a node |
| `min_samples_leaf` | 50 | Requires 50 samples in every leaf node |
| `splitter` | best | Evaluates all features at each split |
| `random_state` | 31 | Reproducibility |

### 3.4 Performance Results (v4)

Train: 692,028 patients &nbsp;•&nbsp; Test: 173,008 patients

| Metric | Train | Test | Interpretation |
|---|---|---|---|
| RMSE | 27.87 months | 28.05 months | Average error, penalizing large misses more |
| MAE | 15.78 months | 15.87 months | Average absolute error ~16 months (median 8) |
| R² | 0.472 | 0.467 | Explains ~47% of variance in survival |
| Overfitting (R² diff) | — | 0.005 | ✓ Negligible |

**Prediction error distribution (test):** median 8.04 months; 75th percentile 16.72; 90th percentile 38.94; 95th percentile 61.45.

> Earlier versions reported R² = 0.752 / MAE = 10.3 months. That result was inflated by target leakage (cause of death used as a feature). See [Changelog](#9-changelog).

### 3.5 Feature Importances (v4)

| Rank | Feature | Importance |
|---|---|---|
| 1 | RX Summ–Surg Prim Site (1998+)_Encoded | 0.5227 |
| 2 | SEER historic stage A (1973–2015)_Encoded | 0.2317 |
| 3 | Age_Mean_Scaled | 0.0898 |
| 4 | Year of diagnosis | 0.0724 |
| 5 | Tumor_Size_mm_Scaled | 0.0370 |
| 6 | Treatment_Days_Scaled | 0.0297 |
| 7 | Sex_Encoded | 0.0148 |
| 8 | Derived Summary Grade 2018 (2018+) with NULL_Encoded | 0.0015 |
| 9 | Marital status at diagnosis_Encoded | 0.0006 |

Surgery and stage dominate, which is consistent with established clinical prognostic factors for lung cancer.

### 3.6 Output Files

*Generated locally when the script runs. CSV and model files are not included in this repository (see [Section 1](#1-data-source)).*

| File | Description |
|---|---|
| `lung_cancer_survival_dt_model.pkl` | Serialized trained Decision Tree model |
| `lung_cancer_survival_predictions.csv` | Actual vs predicted survival months per patient |
| `feature_importance_survival.csv` | Feature importance scores (used to weight NN inputs) |
| `survival_prediction_scatter.png` | Actual vs predicted scatter plot |
| `residuals_plot.png` | Residuals vs predicted values |
| `survival_distributions.png` | Distribution comparison: actual vs predicted |
| `feature_importance_survival.png` | Feature importance bar chart |

### 3.7 Known Warnings (resolved in v4)

| Warning | Fix applied |
|---|---|
| `DtypeWarning` on Derived Summary Grade column | `low_memory=False` in `pd.read_csv()`. Mixed-type parsing had split identical grades into separate codes (16 values instead of 11) |
| `ChainedAssignmentError` on `df[col].fillna(..., inplace=True)` | Replaced with `df[col] = df[col].fillna('Unknown')` |
| `RuntimeWarning: divide by zero` in `Percentage_Error` | Denominator changed to `(actual + 1)` |

---

## 4. Model 2: PyTorch Neural Network

**Script:** `pytorch_neural_network.py`

### 4.1 Purpose

Predicts log-transformed survival months using a feed-forward neural network, benchmarked against the tree-based models. Inputs are pre-weighted by Decision Tree feature importances.

### 4.2 Key Design Choices

- **Target transformation:** `log1p(survival months)` reduces skewness from 3.00 to 0.12, stabilizing training. Predictions are converted back with `expm1()`.
- **Feature weighting (Step 12b):** each input column is multiplied by `sqrt(importance / mean_importance)`, using the v4 Decision Tree importances.
- **Validation set (v4):** 10% of the training data is held out as a validation set to drive the learning-rate scheduler. The test set is used only for final evaluation.

### 4.3 Architecture

| Layer | Size | Components |
|---|---|---|
| Input | 9 features | Weighted feature matrix (`X_nn`) |
| Hidden 1 | 128 neurons | Linear → BatchNorm1d → ReLU → Dropout(0.3) |
| Hidden 2 | 64 neurons | Linear → BatchNorm1d → ReLU → Dropout(0.3) |
| Hidden 3 | 32 neurons | Linear → BatchNorm1d → ReLU → Dropout(0.2) |
| Hidden 4 | 16 neurons | Linear → ReLU |
| Output | 1 neuron | Predicted log(survival months) |
| Total parameters | 12,609 | — |

### 4.4 Training Configuration

| Parameter | Value |
|---|---|
| Loss function | `HuberLoss(delta=1.0)`, robust to outliers |
| Optimizer | `Adam(lr=0.001)` |
| Scheduler | `ReduceLROnPlateau(patience=20, factor=0.5)`, stepped on **validation** loss |
| Gradient clipping | `max_norm=1.0` |
| Epochs | 500 (full-batch) |
| Random state | 31 |

### 4.5 Importance Weights Applied to NN Inputs (v4)

| Feature | Raw DT Importance | Sqrt-Scaled Weight |
|---|---|---|
| RX Summ–Surg Prim Site (1998+)_Encoded | 0.5226 | 2.1688 |
| SEER historic stage A (1973–2015)_Encoded | 0.2317 | 1.4442 |
| Age_Mean | 0.0898 | 0.8989 |
| Year of diagnosis | 0.0723 | 0.8066 |
| Tumor_Size_mm | 0.0370 | 0.5767 |
| Treatment_Days | 0.0297 | 0.5166 |
| Sex_Encoded | 0.0148 | 0.3647 |
| Derived Summary Grade 2018 (2018+) with NULL_Encoded | 0.0016 | 0.1194 |
| Marital status at diagnosis_Encoded | 0.0006 | 0.0712 |

### 4.6 Performance Results (v4)

Train: 622,825 patients &nbsp;•&nbsp; Validation: 69,203 &nbsp;•&nbsp; Test: 173,008

| Metric | Train | Test | Notes |
|---|---|---|---|
| RMSE | 44.20 months | 44.11 months | Worse than the Decision Tree (28.05) |
| MAE | 22.28 months | 22.15 months | Median absolute error 6.82; P90 63.51; P95 102.58 months |
| R² | −0.322 | −0.319 | Worse than predicting the mean for every patient |
| Overfitting (R² diff) | — | −0.003 | Not overfitting; the model is under-fitting / mis-calibrated |

**Training log (selected epochs):**

```
Epoch [50/500]   Train Loss: 0.667733  Validation Loss: 1.400195  LR: 0.001000
Epoch [100/500]  Train Loss: 0.558974  Validation Loss: 0.784229  LR: 0.001000
Epoch [200/500]  Train Loss: 0.528844  Validation Loss: 1.203822  LR: 0.001000
Epoch [300/500]  Train Loss: 0.511715  Validation Loss: 1.085508  LR: 0.001000
Epoch [400/500]  Train Loss: 0.498398  Validation Loss: 1.236885  LR: 0.001000
Epoch [500/500]  Train Loss: 0.483734  Validation Loss: 1.105849  LR: 0.001000
Unique log-scale predictions: 639
Predicted survival (months): min=1.33  max=3.72  mean=1.67  median=1.59
Actual survival (months):    min=0.00  max=275.00  mean=23.32  median=9.00
```

**What the results show**

- Every patient is predicted to survive 1.3–3.7 months, below even the median actual survival of 9 months. This is a systematic downward bias, not only a narrow range.
- Validation loss stays roughly double the training loss and bounces between checkpoints without improving, and the learning rate never decreased.
- Removing cause of death made little difference (v3: R² −0.286), so leakage was not what held this model back.

**Most likely cause (moderate confidence):** a train/eval-mode mismatch. `BatchNorm1d` uses batch statistics in `model.train()` and running averages in `model.eval()`. Year of diagnosis enters the network unscaled (values ≈2000–2022) while the other inputs are near zero, and Step 12b rescales columns further, so the running averages may not reflect what the model learned from. Full-batch training (only 500 gradient steps in total) and label-encoded categoricals are likely secondary contributors.

**Next diagnostic steps, in order:**

1. Standardize Year of diagnosis along with the other numeric inputs.
2. Remove the Step 12b importance weighting.
3. Compare `model.train()` and `model.eval()` predictions on the same rows to confirm or rule out the BatchNorm hypothesis.
4. Switch to mini-batch training with a `DataLoader` (e.g. batch size 1,024).
5. One-hot encode (or embed) categorical features.

> Tree-based models often outperform neural networks on tabular data, so the NN may still trail the Decision Tree after these fixes. It is kept as a documented benchmark, not a production model.

### 4.7 Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| All predictions identical | Importance weights shrink low-importance features toward zero | Use `sqrt()` scaling, not linear normalization; or remove weighting |
| Negative R² (current v4 state) | Model performs worse than predicting the mean | Standardize all numeric inputs, remove Step 12b weighting, use mini-batches |
| Validation loss ~2x training loss and unstable | Likely BatchNorm train/eval mismatch with unscaled inputs | Standardize Year of diagnosis; compare `model.train()` vs `model.eval()` outputs |
| <10 unique log predictions | Model collapsed to constant output | Check Step 12b weight calculation |
| Loss oscillating wildly | Learning rate too high | Reduce `lr` from `0.01` to `0.001` |
| `ImportError: DLL load failed while importing _C` | Broken or conflicting PyTorch install on Windows | Reinstall in a clean environment (see [Section 7](#7-environment-setup)) |

### 4.8 Output Files

*Generated locally when the script runs. CSV and model files are not included in this repository (see [Section 1](#1-data-source)).*

| File | Description |
|---|---|
| `lung_cancer_survival_nn_log.pth` | Trained network weights (PyTorch state dict) |
| `nn_lung_cancer_predictions_log.csv` | Patient-level predictions with absolute and percentage error |
| `log_transformation_comparison.png` | Before/after histogram showing skewness reduction |
| `training_loss_curve_log.png` | Training and validation Huber loss over 500 epochs |
| `nn_actual_vs_predicted_log.png` | Actual vs predicted survival months |
| `nn_residuals_plot_log.png` | Residuals vs predicted values |
| `nn_distributions_comparison_log.png` | Distribution of actual vs predicted survival |

---

## 5. Model 3: Random Survival Forest

**Script:** `random_survival_forest.py`

### 5.1 Purpose

Models the full survival distribution rather than a single time point. Unlike the Decision Tree and Neural Network, the RSF accounts for **censoring** (patients still alive at last follow-up). It outputs a risk score and a survival curve per patient, evaluated with the concordance index (C-index).

### 5.2 Key Conceptual Differences

| Aspect | Decision Tree / Neural Network | Random Survival Forest |
|---|---|---|
| Target | Survival months | Structured array: (event occurred, time) |
| Censoring | Not modeled | Patients marked `Alive` treated as censored |
| Output | Predicted months | Risk score + survival probability curve |
| Primary metric | R² / RMSE / MAE | Concordance index (C-index) |
| Feature importance | Impurity-based (built-in) | Permutation importance (C-index drop when shuffled) |
| Feature weighting | DT importances applied to NN inputs | Unweighted features |

### 5.3 Event Indicator (Censoring)

The cause-of-death column defines the outcome. It is **not** used as a model feature.

- **Event = True:** cause of death is `Lung And Bronchus` (598,829 patients, 78.0%)
- **Event = False:** `Alive` at last follow-up, censored (168,708 patients, 22.0%)
- Rows with 0-month survival are removed (RSF requires positive survival times)

This is a **cause-specific** approach. A competing-risks approach (Fine-Gray, Aalen-Johansen cumulative incidence) would be the next refinement once other-cause deaths are restored to the cohort (see [Section 8](#8-known-limitations--to-do)).

### 5.4 Model Parameters

| Parameter | Value | Rationale |
|---|---|---|
| `n_estimators` | 100 | Number of trees |
| `max_depth` | 15 | Controls tree complexity |
| `min_samples_split` | 50 | Minimum samples to split a node |
| `min_samples_leaf` | 25 | Minimum samples in a leaf |
| `max_features` | `sqrt` | Standard random forest heuristic |
| `n_jobs` | -1 | Use all CPU cores for training |
| `random_state` | 31 | Reproducibility |

### 5.5 Understanding the C-index

The C-index answers: *if two patients are picked at random, how often does the model correctly identify which one will die sooner?*

| C-index Value | Interpretation |
|---|---|
| 1.0 | Perfect ranking |
| **0.72 (this model)** | **Correct ~72% of the time** |
| 0.5 | Random, equivalent to a coin flip |
| < 0.5 | Worse than random |

### 5.6 Performance Results (v4)

Train: 614,029 patients &nbsp;•&nbsp; Test: 153,508 patients

| Metric | Train (10k sample) | Test (10k sample) | Notes |
|---|---|---|---|
| C-index | 0.7226 | 0.7206 | Near-identical, no overfitting |
| Random baseline | 0.5000 | 0.5000 | Coin flip comparison |

> Earlier versions reported C-index = 0.761. That result was inflated because cause of death both defined the event and was used as a feature.

### 5.7 Memory Management

The RSF allocates an `(n_patients × n_timepoints × 2)` array internally. Three mitigations keep it within memory:

- **Batched prediction (5,000 patients per batch):** `rsf_model.predict()` is never called on the full dataset at once.
- **C-index on 10,000-patient samples:** `concordance_index_censored()` is O(n²) and infeasible on 614k patients.
- **Permutation importance on a 5,000-patient subsample with `n_jobs=1`:** avoids parallel memory errors.

### 5.8 Feature Importance (Permutation, v4)

| Rank | Feature | Importance | Std |
|---|---|---|---|
| 1 | RX Summ–Surg Prim Site (1998+)_Encoded | 0.0591 | ±0.0017 |
| 2 | SEER historic stage A (1973–2015)_Encoded | 0.0249 | ±0.0011 |
| 3 | Age_Mean | 0.0173 | ±0.0016 |
| 4 | Year of diagnosis | 0.0169 | ±0.0011 |
| 5 | Tumor_Size_mm | 0.0157 | ±0.0005 |
| 6 | Treatment_Days | 0.0134 | ±0.0009 |
| 7 | Sex_Encoded | 0.0025 | ±0.0004 |
| 8 | Marital status at diagnosis_Encoded | 0.0007 | ±0.0003 |
| 9 | Derived Summary Grade 2018 (2018+) with NULL_Encoded | −0.0022 | ±0.0002 |

The top three predictors match the Decision Tree (surgery, stage, age), so two different methods agree. Tumor grade is slightly negative (noise), likely because it is only recorded from 2018 onward.

### 5.9 Risk Quartile Analysis (v4)

| Quartile | N | Event % | Median Survival | Mean Survival | IQR (P25–P75) |
|---|---|---|---|---|---|
| Q1 (Lowest Risk) | 38,377 | 33.6% | 31.0 months | 50.6 months | 12–71 months |
| Q2 | 38,379 | 82.0% | 13.0 months | 28.1 months | 5–34 months |
| Q3 | 38,375 | 97.4% | 9.0 months | 17.1 months | 4–19 months |
| Q4 (Highest Risk) | 38,377 | 99.3% | 5.0 months | 9.6 months | 2–11 months |

The median survival gap between Q1 and Q4 is **26 months** (31 vs 5), and the lung cancer death rate rises steadily from 34% to 99%. This shows the risk score **discriminates** well (ranks patients correctly). It does not by itself establish **calibration** (whether predicted probabilities match observed rates).

### 5.10 Output Files

*Generated locally when the script runs. CSV and model files are not included in this repository (see [Section 1](#1-data-source)).*

| File | Description |
|---|---|
| `lung_cancer_rsf_model.pkl` | Trained RSF model |
| `rsf_predictions.csv` | Patient-level risk scores, event status, and risk group |
| `rsf_feature_importance.csv` | Permutation importances with standard deviation |
| `rsf_quartile_summary.csv` | Survival statistics per risk quartile |
| `rsf_risk_score_distribution.png` | Risk score density by outcome |
| `rsf_risk_vs_survival.png` | Risk score vs actual survival months |
| `rsf_survival_curves.png` | Average survival curves: high vs low risk |
| `rsf_feature_importance.png` | Permutation importance bar chart |
| `rsf_quartile_analysis.png` | Median survival and event rate per quartile |

---

## 6. Model Comparison

| Aspect | Decision Tree | Neural Network | Random Survival Forest |
|---|---|---|---|
| Best use case | Interpretable baseline; feature importance source | Benchmark for nonlinear patterns | Survival ranking; censoring; risk stratification |
| Primary metric | R² = 0.467 | R² = −0.319 (underperforms) | C-index = 0.721 |
| MAE | 15.87 months | 22.15 months (predictions compressed to 1.33–3.72) | N/A (risk score output) |
| Handles censoring | No | No | Yes |
| Predicts exact months | Yes | Yes (unreliable currently) | No (risk score + survival curve) |
| Training time | ~1 min | ~5–10 min | ~20–40 min |
| Memory requirements | Low | Moderate | High (batched prediction required) |
| Missing value handling | Native (scikit-learn ≥ 1.3) | Median imputation | Median imputation |

### 6.1 Current Status

| Model | Status | Notes |
|---|---|---|
| Decision Tree | ✅ Complete (v4) | R² = 0.467 with no overfitting; clinically sensible feature ranking |
| Random Survival Forest | ✅ Complete (v4) | C-index = 0.721; strongest model for risk stratification |
| Neural Network | ⚠️ Benchmark, needs work | R² = −0.319; likely train/eval-mode mismatch (see [Section 4.6](#46-performance-results-v4)). Trees outperform it on this tabular data |

---

## 7. Environment Setup

### 7.1 Conda Environment

Run all three scripts in a dedicated environment. A clean environment also avoids Windows DLL conflicts with PyTorch.

```bash
conda create -n seer python=3.12 -y
conda activate seer

# PyTorch (CPU build), installed with pip per PyTorch's current guidance
pip install torch --index-url https://download.pytorch.org/whl/cpu

# Everything else
conda install -c conda-forge scikit-survival scikit-learn pandas numpy matplotlib seaborn -y
```

### 7.2 Key Dependencies

| Package | Purpose |
|---|---|
| `torch` | Neural network (PyTorch) |
| `scikit-survival` | Random Survival Forest |
| `scikit-learn ≥ 1.3` | Decision Tree with native NaN support |
| `pandas ≥ 2.0` | Data manipulation |
| `numpy` | Numeric operations |
| `matplotlib` / `seaborn` | Visualizations |

### 7.3 Running the Scripts

```bash
conda activate seer

python decision_tree_regressor.py
python pytorch_neural_network.py
python random_survival_forest.py
```

Run the Decision Tree first: the Neural Network uses its feature importances.

### 7.4 Expected Runtimes

| Script | Approximate Runtime | Bottleneck |
|---|---|---|
| Decision Tree | 1–2 minutes | Training on 692k patients |
| Neural Network | 5–10 minutes | 500 full-batch epochs |
| Random Survival Forest | 20–40 minutes | Training + batched prediction on 614k patients |

---

## 8. Known Limitations / To-Do

- [ ] **Re-extract the SEER cohort to include all causes of death.** The current SQL output contains only `Lung And Bronchus` and `Alive`, so patients who died of other causes were filtered out. This is selection bias: excluded patients likely skew older and sicker. Next step: re-run the extraction with all cause-of-death values, then either censor other-cause deaths (cause-specific) or model them as a competing risk (Fine-Gray / Aalen-Johansen cumulative incidence).
- [x] ~~Rerun the Neural Network with cause of death removed, v4 importance weights, and the validation set.~~ Done: R² = −0.319 (Section 4.6).
- [ ] **Fix the Neural Network:** standardize Year of diagnosis, remove Step 12b weighting, check train vs eval mode, add mini-batching (Section 4.6).
- [x] ~~Add `low_memory=False` to the Decision Tree script.~~ Done (Section 3.7).
- [ ] **Add RSF calibration checks** (predicted vs observed survival at 12 and 24 months); the C-index measures ranking only.
- [ ] **Switch Neural Network categorical inputs from label encoding to one-hot encoding.** Label encoding implies an order (e.g. marital status 0–6) that neural networks treat as meaningful.
- [ ] **Fit scaler and median imputation on the training set only**, then apply to the test set, to remove minor preprocessing leakage.
- [ ] **Investigate year of diagnosis.** Part of its signal may reflect shorter possible follow-up for recent diagnoses rather than true survival differences.
- [ ] **Consider dropping tumor grade**, which shows negative permutation importance and is only recorded 2018+.
- [ ] **Decision Tree and Neural Network do not handle censoring.** Treating still-alive patients' months as survival times biases predictions downward; the RSF is the appropriate model for survival questions.

---

## 9. Changelog

**v4 (October 2026)**
- **Removed cause of death as a feature in all three models** (target leakage: it is only known after the outcome). In the RSF it is now used only to define the event.
  - Decision Tree test R²: 0.752 → **0.467**; MAE: 10.3 → **15.9 months**
  - RSF test C-index: 0.761 → **0.721**
- Neural Network: updated importance weights from the v4 Decision Tree; added a validation set for the learning-rate scheduler.
  - Test R²: −0.286 → **−0.319**; MAE: 21.6 → **22.2 months** (still underperforms; diagnosis in Section 4.6)
- Fixed data file paths, `fillna` chained-assignment warning, and divide-by-zero in percentage error; added `low_memory=False` to all three scripts.
- Renamed scripts to standard `.py` filenames (`decision_tree_regressor.py`, `pytorch_neural_network.py`, `random_survival_forest.py`).
- Moved to the `seer` conda environment (Python 3.12) with CPU PyTorch to resolve the Windows DLL import error.
- Corrected RSF interpretation text: a rising event rate across quartiles shows discrimination, not calibration.

**v3**
- Median imputation restored; importance weighting applied to NN inputs only; permutation importance for RSF; sqrt-scaled weights to prevent gradient collapse.

---

*SEER citation: Surveillance, Epidemiology, and End Results (SEER) Program (www.seer.cancer.gov) SEER\*Stat Database: Incidence – SEER Research Data, 17 Registries, Nov 2024 Sub (2000–2022), National Cancer Institute, DCCPS, Surveillance Research Program, released April 2025.*