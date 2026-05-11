# SEER Lung Cancer Survival Analysis
### Machine Learning Model Documentation

> **Data:** SEER*Stat Version 9.0.43.0 &nbsp;•&nbsp; Extracted May 10, 2026 &nbsp;•&nbsp; 17 Registries, 2000–2022

---

## Table of Contents
1. [Data Source](#1-data-source)
2. [Shared Preprocessing](#2-shared-preprocessing-steps)
3. [Model 1: Decision Tree Regressor](#3-model-1-decision-tree-regressor)
4. [Model 2: PyTorch Neural Network](#4-model-2-pytorch-neural-network)
5. [Model 3: Random Survival Forest](#5-model-3-random-survival-forest)
6. [Model Comparison](#6-model-comparison)
7. [Environment Setup](#7-environment-setup)

---

## 1. Data Source

This analysis uses data extracted from the Surveillance, Epidemiology, and End Results (SEER) Program, maintained by the National Cancer Institute. The dataset covers malignant lung and bronchus cancer cases across 17 registries from 2000 to 2022.

> **Note:** `SEER_LUNG_11_14_EXT.csv` is a truncated example file containing only 2011 - 2014 records. The full analysis requires the complete SEER SQL export.

### 1.1 Citation

Surveillance, Epidemiology, and End Results (SEER) Program (www.seer.cancer.gov) SEER\*Stat Database: Incidence – SEER Research Data, 17 Registries, Nov 2024 Sub (2000–2022) – Linked To County Attributes – Time Dependent (1990–2023) Income/Rurality, 1969–2023 Counties, National Cancer Institute, DCCPS, Surveillance Research Program, released April 2025, based on the November 2024 submission.

### 1.2 Dataset Overview

| Property | Value |
|---|---|
| Selection Criteria | Malignant Behavior, Known Age | {Site and Morphology>Site recode ICD-O-3 2023 Revision Expanded} = ' Lung And Bronchus |
| Total records extracted | 1,105,791 |
| Records after exclusions | 887,217 |
| Records with valid survival data | 865,036 |
| Records used in RSF (after removing 0-month survival) | 767,537 |
| Diagnosis years | 2000–2022 |
| Registries | 17 (SEER Research Data) |
| Example file (truncated) | SEER_LUNG_11_14_EXT.csv (2014 only) |

### 1.3 Features Used Across All Models

| Column Name | Type | Description |
|---|---|---|
| Patient ID | Identifier | Unique patient identifier |
| Age recode with <1 year olds and 90+ | Categorical → Numeric | Age group; converted to midpoint (`Age_Mean`) |
| Sex | Categorical | Patient sex (2 unique values) |
| Marital status at diagnosis | Categorical | Marital status (7 unique values) |
| Year of diagnosis | Numeric | Calendar year of diagnosis |
| SEER historic stage A (1973–2015) | Categorical | Cancer stage (5 unique values) |
| Derived Summary Grade 2018 (2018+) with NULL | Categorical | Tumor grade (11–16 unique values; high missingness) |
| RX Summ–Surg Prim Site (1998+) | Categorical | Surgical treatment code (25 unique values) |
| Time from diagnosis to treatment in days recode with NULL | Categorical → Numeric | Days to treatment; high missingness (~37%) |
| Tumor Size Over Time Recode (1988+) with NULL | Categorical → Numeric | Tumor size in mm; high missingness (~29%) |
| Survival months with NULL | Numeric | Months survived — target variable for DT and NN |
| COD to site recode ICD-O-3 2023 Revision Expanded (1999+) | Categorical | Cause of death; used as event indicator in RSF |

---

## 2. Shared Preprocessing Steps

All three models share the following preprocessing pipeline before model-specific steps diverge.

| Step | Operation | Notes |
|---|---|---|
| Deduplication | `df.drop_duplicates()` | Removes ~235 duplicate rows |
| Target filtering | Drop rows where survival months is null | Retains 865,036 patients |
| Age engineering | Extract midpoint from age range strings | E.g. `"65-69 years"` → 67.0; `"90+"` → 92 |
| Tumor size | Extract numeric mm from string field | NULL/Unknown → NaN |
| Treatment days | Extract integer from string field | NULL/Unknown → NaN |
| Categorical encoding | `LabelEncoder` per column; NaN → `'Unknown'` | Applied before encoding |
| Missing imputation | Median fill for `Age_Mean`, `Tumor_Size_mm`, `Treatment_Days` | Decision Tree uses native missing value handling instead |
| Standardization | `StandardScaler` (mean=0, std=1) on numeric features | Applied after imputation |

---

## 3. Model 1: Decision Tree Regressor

**Script:** `Python Decision Tree Classifier`

### 3.1 Purpose

Predicts exact survival months as a continuous regression target. Serves as the baseline model and the source of feature importances used to weight inputs to the Neural Network.

### 3.2 Key Differences from Other Models

- **Target:** raw survival months (regression, not ranking or log-transformed)
- **Missing values** handled natively by scikit-learn ≥ 1.3 — no imputation required for `Tumor_Size_mm` or `Treatment_Days`
- Feature columns use `_Scaled` suffix (`Age_Mean_Scaled`, `Tumor_Size_mm_Scaled`, `Treatment_Days_Scaled`)

### 3.3 Model Parameters

| Parameter | Value | Rationale |
|---|---|---|
| `max_depth` | 10 | Limits tree depth to prevent overfitting |
| `min_samples_split` | 100 | Requires 100 samples before splitting a node |
| `min_samples_leaf` | 50 | Requires 50 samples in every leaf node |
| `splitter` | best | Evaluates all features at each split |
| `random_state` | 31 | Reproducibility |

### 3.4 Expected Terminal Output

```
Training set: 692,028 patients
Test set:     173,008 patients
Training complete!

Test Set Performance:
  RMSE: ~19 months
  MAE:  ~10 months
  R² Score: ~0.75
```

### 3.5 Performance Results

| Metric | Train | Test | Interpretation |
|---|---|---|---|
| RMSE | 18.96 months | 19.11 months | Average prediction error (penalises large errors more) |
| MAE | 10.32 months | 10.34 months | Typical patient is off by ~10 months |
| R² Score | 0.756 | 0.752 | Model explains 75.2% of survival variance |
| Overfitting (R² diff) | — | 0.003 | ✓ Negligible — model generalises well |

### 3.6 Feature Importances

| Rank | Feature | Importance Score |
|---|---|---|
| 1 | Year of diagnosis | 0.5429 |
| 2 | COD to site recode ICD-O-3 2023 Revision Expanded (1999+)_Encoded | 0.3076 |
| 3 | RX Summ–Surg Prim Site (1998+)_Encoded | 0.0783 |
| 4 | SEER historic stage A (1973–2015)_Encoded | 0.0272 |
| 5 | Treatment_Days_Scaled | 0.0137 |
| 6 | Derived Summary Grade 2018 (2018+) with NULL_Encoded | 0.0115 |
| 7 | Age_Mean_Scaled | 0.0081 |
| 8 | Tumor_Size_mm_Scaled | 0.0075 |
| 9 | Sex_Encoded | 0.0025 |
| 10 | Marital status at diagnosis_Encoded | 0.0006 |

### 3.7 Output Files

| File | Description |
|---|---|
| `lung_cancer_survival_dt_model.pkl` | Serialised trained Decision Tree model |
| `lung_cancer_survival_predictions.csv` | Actual vs predicted survival months per patient |
| `feature_importance_survival.csv` | Feature importance scores used to weight NN inputs |
| `survival_prediction_scatter.png` | Actual vs predicted scatter plot |
| `residuals_plot.png` | Residuals vs predicted values |
| `survival_distributions.png` | Distribution comparison: actual vs predicted |
| `feature_importance_survival.png` | Top 10 feature importance bar chart |

### 3.8 Known Warnings

| Warning | Fix |
|---|---|
| `DtypeWarning` on Derived Summary Grade column | Add `low_memory=False` to `pd.read_csv()` |
| `ChainedAssignmentError` on `df[col].fillna(..., inplace=True)` | Replace with `df[col] = df[col].fillna('Unknown')` |
| `RuntimeWarning: divide by zero` in Percentage_Error column | Cosmetic only — occurs when actual survival = 0; does not affect model |

---

## 4. Model 2: PyTorch Neural Network

**Script:** `Pytorch Neural Network`

### 4.1 Purpose

Predicts log-transformed survival months using a deep feed forward neural network. Inputs are pre-weighted by Decision Tree feature importances to emphasise the most predictive features before training.

### 4.2 Key Design Choices

- **Target transformation:** `log1p(survival months)` — reduces skewness from 3.00 to 0.12, stabilising training. Predictions are transformed back with `expm1()` for interpretability.
- **Feature weighting (Step 12b):** each input column is multiplied by `sqrt(importance / mean_importance)`, boosting Year of diagnosis (~2.3×) while avoiding gradient collapse from linear scaling (~5.3×).
- The RSF receives the original **unweighted** feature matrix — weighting applies to the NN only.

### 4.3 Architecture

| Layer | Size | Components |
|---|---|---|
| Input | 10 features | Weighted feature matrix (`X_nn`) |
| Hidden 1 | 128 neurons | Linear → BatchNorm1d → ReLU → Dropout(0.3) |
| Hidden 2 | 64 neurons | Linear → BatchNorm1d → ReLU → Dropout(0.3) |
| Hidden 3 | 32 neurons | Linear → BatchNorm1d → ReLU → Dropout(0.2) |
| Hidden 4 | 16 neurons | Linear → ReLU |
| Output | 1 neuron | Predicted log(survival months) |
| Total parameters | 12,737 | — |

### 4.4 Training Configuration

| Parameter | Value |
|---|---|
| Loss function | `HuberLoss(delta=1.0)` — robust to outliers |
| Optimizer | `Adam(lr=0.001)` |
| Scheduler | `ReduceLROnPlateau(patience=20, factor=0.5)` |
| Gradient clipping | `max_norm=1.0` |
| Epochs | 500 |
| Random state | 31 |

### 4.5 Importance Weights Applied to NN Inputs

These weights are derived from Decision Tree feature importances and applied in Step 12b. The `sqrt()` transformation prevents Year of diagnosis from dominating gradients.

| Feature | Raw DT Importance | Sqrt-Scaled Weight Applied |
|---|---|---|
| Year of diagnosis | 0.5429 | 2.2953 |
| COD to site recode ICD-O-3 2023 Revision Expanded (1999+)_Encoded | 0.3076 | 1.7391 |
| RX Summ–Surg Prim Site (1998+)_Encoded | 0.0783 | 0.8835 |
| SEER historic stage A (1973–2015)_Encoded | 0.0272 | 0.5410 |
| Treatment_Days | 0.0137 | 0.4417 |
| Derived Summary Grade 2018 (2018+) with NULL_Encoded | 0.0115 | 0.3825 |
| Age_Mean | 0.0081 | 0.3561 |
| Tumor_Size_mm | 0.0075 | 0.3422 |
| Sex_Encoded | 0.0025 | 0.1711 |
| Marital status at diagnosis_Encoded | 0.0006 | 0.1397 |

### 4.6 Expected Terminal Output

```
Epoch [50/500]   Train Loss: ~1.06  Test Loss: ~1.49  LR: 0.001000
Epoch [100/500]  Train Loss: ~0.55  Test Loss: ~1.04  LR: 0.001000
...
Epoch [500/500]  Train Loss: ~0.43  Test Loss: ~1.38  LR: 0.001000
Training complete!
Unique log-scale predictions: 807  ✓ Predictions vary as expected
```

### 4.7 Troubleshooting: Constant Predictions

If all patients receive the same predicted value (e.g. `4.44 months` for all), the model has failed to learn.

| Symptom | Cause | Fix |
|---|---|---|
| All predictions identical | Importance weights crush low-importance features to near-zero | Use `sqrt()` scaling, not linear normalisation |
| Negative R² (−0.3 to −0.4) | Model performs worse than predicting the mean | Verify Step 12b weighting; reduce learning rate to 0.001 |
| <10 unique log predictions | Model collapsed to constant output | Check Step 12b weight calculation |
| Test loss oscillating wildly | Learning rate too high | Reduce `lr` from `0.01` to `0.001` |

### 4.8 Output Files

| File | Description |
|---|---|
| `lung_cancer_survival_nn_log.pth` | Trained neural network weights (PyTorch state dict) |
| `nn_lung_cancer_predictions_log.csv` | Patient-level predictions with absolute and percentage error |
| `nn_model_summary_log.txt` | Model architecture, parameters, and performance metrics |
| `log_transformation_comparison.png` | Before/after histogram showing skewness reduction |
| `training_loss_curve_log.png` | Train and test Huber loss over 500 epochs |
| `nn_actual_vs_predicted_log.png` | Scatter plot of actual vs predicted survival months |
| `nn_residuals_plot_log.png` | Residuals vs predicted values (bias check) |
| `nn_distributions_comparison_log.png` | Distribution of actual vs predicted survival |

---

## 5. Model 3: Random Survival Forest

**Script:** `Random Survival Forest`

### 5.1 Purpose

Models the full survival distribution rather than predicting a single time point. Unlike the Decision Tree and Neural Network (which predict months as a regression target), the RSF accounts for **censoring** — patients who are still alive at the time of data collection. It outputs a risk score and survival probability curve per patient, evaluated using the concordance index (C-index).

### 5.2 Key Conceptual Differences

| Aspect | Decision Tree / Neural Network | Random Survival Forest |
|---|---|---|
| Target | Survival months (exact number) | Structured array: (event occurred, time) |
| Censoring | Not modelled | Patients marked `'Alive'` treated as censored |
| Output | Predicted months | Risk score + survival probability curve |
| Primary metric | R² / RMSE / MAE | Concordance index (C-index) |
| Feature importance | Impurity-based (built-in) | Permutation importance (C-index drop when shuffled) |
| Feature weighting | DT importances applied to NN inputs | Original unweighted features (weighting distorts splits) |

### 5.3 Event Indicator (Censoring)

The RSF requires knowing whether each patient's event (death from lung cancer) was observed or censored. This is derived from the COD column:

- **Event = True:** COD contains `'Lung And Bronchus'` — patient died of lung cancer (598,829 patients, 78.0%)
- **Event = False:** COD = `'Alive'` — patient was still alive at last follow-up, censored (168,708 patients, 22.0%)
- Rows with 0-month survival are removed (not valid for RSF log-rank calculations)

### 5.4 Model Parameters

| Parameter | Value | Rationale |
|---|---|---|
| `n_estimators` | 100 | Number of trees in the forest |
| `max_depth` | 15 | Controls tree complexity |
| `min_samples_split` | 50 | Minimum samples required to split a node |
| `min_samples_leaf` | 25 | Minimum samples in leaf nodes |
| `max_features` | `sqrt` | Standard random forest heuristic |
| `n_jobs` | -1 | Use all available CPU cores for training |
| `random_state` | 31 | Reproducibility |

### 5.5 Understanding the C-index

The concordance index (C-index) answers: *if two patients are picked at random, how often does the model correctly identify which one will die sooner?*

| C-index Value | Interpretation |
|---|---|
| 1.0 | Perfect — always correctly ranks which patient dies first |
| **0.76 (this model)** | **Correct 76% of the time — good discrimination** |
| 0.70–0.80 | Typical range for published lung cancer survival models |
| 0.5 | Random — equivalent to a coin flip |
| < 0.5 | Worse than random |

### 5.6 Performance Results

| Metric | Train (10k sample) | Test (10k sample) | Notes |
|---|---|---|---|
| C-index | 0.7588 | 0.7607 | Near-identical — no overfitting |
| Random baseline | 0.5000 | 0.5000 | Coin flip comparison point |
| Improvement over random | +0.26 | +0.26 | Model discriminates well |

### 5.7 Memory Management

The RSF requires special handling on large datasets due to its internal `(n_patients × n_timepoints × 2)` array allocation. Three key mitigations are applied:

- **Batched prediction (5,000 patients per batch):** `rsf_model.predict()` is never called on the full dataset at once. Peak memory per batch: ~22 MB vs 2.52 GB for the full test set.
- **C-index computed on 10,000-patient samples:** `concordance_index_censored()` has O(n²) complexity — infeasible on 614k patients. A random sample gives a statistically representative estimate.
- **Permutation importance on 5,000-patient subsample with `n_jobs=1`:** avoids parallel memory allocation errors seen with larger samples.

### 5.8 RSF Feature Importance (Permutation)

Because `RandomSurvivalForest` does not support `.feature_importances_`, permutation importance is used instead. Each feature is randomly shuffled and the drop in C-index is measured. Positive values indicate the feature contributes to correct ranking; negative values indicate noise.

| Rank | Feature | Importance | Std |
|---|---|---|---|
| 1 | COD to site recode ICD-O-3 2023 Revision Expanded (1999+)_Encoded | 0.1367 | ±0.0030 |
| 2 | RX Summ–Surg Prim Site (1998+)_Encoded | 0.0303 | ±0.0024 |
| 3 | SEER historic stage A (1973–2015)_Encoded | 0.0267 | ±0.0016 |
| 4 | Treatment_Days | 0.0161 | ±0.0017 |
| 5 | Tumor_Size_mm | 0.0070 | ±0.0015 |
| 6 | Sex_Encoded | 0.0020 | ±0.0001 |
| 7 | Age_Mean | 0.0017 | ±0.0014 |
| 8 | Derived Summary Grade 2018 (2018+) with NULL_Encoded | −0.0000 | ±0.0003 |
| 9 | Marital status at diagnosis_Encoded | −0.0010 | ±0.0007 |
| 10 | Year of diagnosis | −0.0061 | ±0.0003 |

> **Note on Year of diagnosis:** This feature ranks #1 in the Decision Tree (importance 0.54) but is slightly negative in the RSF (−0.006). The RSF captures the strong clinical signal from COD, stage, and surgical treatment directly, while the Decision Tree's impurity measure is sensitive to the year trend in the data.

### 5.9 Risk Quartile Analysis

Patients are binned into four equal-sized groups by risk score, translating abstract scores into clinically interpretable survival windows.

| Quartile | N | Event% | Median Survival | Mean Survival | IQR (P25–P75) |
|---|---|---|---|---|---|
| Q1 (Lowest Risk) | 38,377 | 12.3% | 43.0 months | 61.6 months | 16–89 months |
| Q2 | 38,377 | 100.0% | 13.0 months | 22.7 months | 6–29 months |
| Q3 | 38,377 | 100.0% | 7.0 months | 11.9 months | 3–15 months |
| Q4 (Highest Risk) | 38,377 | 100.0% | 5.0 months | 9.3 months | 2–11 months |

The median survival gap between Q1 and Q4 is **38 months** (43 vs 5). Q1's 12.3% event rate confirms the low-risk group is predominantly patients who were alive at last follow-up (censored). Q2–Q4 have 100% event rates, indicating the RSF has effectively separated surviving patients from those who died of lung cancer.

### 5.10 Output Files

| File | Description |
|---|---|
| `lung_cancer_rsf_model.pkl` | Trained RSF model (serialised with pickle) |
| `rsf_predictions.csv` | Patient-level risk scores, event status, and risk group (High/Low) |
| `rsf_feature_importance.csv` | Permutation importance scores with standard deviation |
| `rsf_quartile_summary.csv` | Survival statistics per risk quartile |
| `rsf_risk_score_distribution.png` | Risk score density by outcome (died vs censored) |
| `rsf_risk_vs_survival.png` | Risk score vs actual survival months scatter (coloured by event) |
| `rsf_survival_curves.png` | Average survival probability curves: high vs low risk groups |
| `rsf_feature_importance.png` | Permutation importance bar chart with error bars |
| `rsf_quartile_analysis.png` | Median survival and event rate per risk quartile |

---

## 6. Model Comparison

| Aspect | Decision Tree | Neural Network | Random Survival Forest |
|---|---|---|---|
| Best use case | Interpretable baseline; feature importance source | Complex nonlinear patterns with log-scale regularisation | Survival ranking; censoring; clinical risk stratification |
| Primary metric | R² = 0.752 | R² = −0.343 *(needs improvement)* | C-index = 0.761 |
| MAE | 10.34 months | 22.74 months *(near-constant predictions)* | N/A (risk score output) |
| Handles censoring | No | No | Yes |
| Predicts exact months | Yes | Yes *(unreliable currently)* | No (risk score + survival curve) |
| Training time | ~1 min | ~5–10 min | ~15–30 min on 614k patients |
| Memory requirements | Low | Moderate | High — batched prediction required |
| Missing value handling | Native (scikit-learn ≥ 1.3) | Median imputation | Median imputation |
| Feature weighting | Not applicable | DT importance-weighted inputs | Unweighted (preserves splits) |

### 6.1 Current Status and Recommendations

| Model | Status | Recommendation |
|---|---|---|
| Decision Tree | ✅ Production-ready | Best regression model (R² = 0.752, MAE = 10.3 months). Use as primary survival time estimator. |
| Random Survival Forest | ✅ Production-ready | Strongest model for risk stratification and survival curve analysis. C-index 0.761 is competitive with published literature. |
| Neural Network | ⚠️ Needs improvement | Currently underperforming (near-constant predictions, R² < 0). Recommend removing Step 12b importance weighting and retraining without input scaling as the next diagnostic step. |

---

## 7. Environment Setup

### 7.1 Conda Environment

All three scripts should be run in a dedicated environment with the neccesary dependencies outlined below. 


```

### 7.2 Key Dependencies

| Package | Purpose | Install Command |
|---|---|---|
| `torch` | Neural network (PyTorch) | `conda install pytorch -c pytorch` |
| `scikit-survival` | Random Survival Forest | `conda install -c conda-forge scikit-survival` |
| `scikit-learn ≥ 1.3` | Decision Tree with native NaN support | `conda install scikit-learn` |
| `pandas ≥ 2.0` | Data manipulation | `conda install pandas` |
| `numpy` | Numeric operations | `conda install numpy` |
| `matplotlib` / `seaborn` | Visualisations | `conda install matplotlib seaborn` |

### 7.3 Running the Scripts

```bash
# Activate environment first
python activate #env_name

# Decision Tree
python "Python Decision Tree Classifier"

# Neural Network
python "Pytorch Neural Network"

# Random Survival Forest
python "Random Survival Forest"
```

### 7.4 Expected Runtimes

| Script | Approximate Runtime | Bottleneck |
|---|---|---|
| Decision Tree | 1–2 minutes | Training on 692k patients |
| Neural Network | 5–10 minutes | 500 epochs on 692k patients |
| Random Survival Forest | 20–40 minutes | RSF training + batched prediction on 614k patients |

---

*SEER citation: Surveillance, Epidemiology, and End Results (SEER) Program (www.seer.cancer.gov) SEER\*Stat Database: Incidence – SEER Research Data, 17 Registries, Nov 2024 Sub (2000–2022), National Cancer Institute, DCCPS, Surveillance Research Program, released April 2025.*