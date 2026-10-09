# Weak Signal Diagnosis Report

## Project
**Telecom Churn Prediction & Retention Decision System**

## Purpose of this Report
This report documents the diagnostic investigation performed after multiple classification models produced weak validation performance. The objective was to determine whether the poor results were primarily caused by model choice, preprocessing, class imbalance, or limited predictive signal in the dataset.

---

## 1. Dataset Overview

The modeling dataset contained:

- **10,000 customers**
- **41 total columns** in the saved engineered dataset
- **37 model input features** after excluding identifiers, raw date fields, and the target
- **Target:** `Churn`
- **Churn prevalence:** **7.86%**
- **Non-churn prevalence:** **92.14%**

The data included customer demographics, contract type, monthly charges, tenure, monthly usage behavior, call/data/SMS/complaint aggregates, recent-vs-early behavioral change features, 12-month trend slopes, and data-quality indicator variables.

A stratified train/validation/test split was used:

- **Train:** 70%
- **Validation:** 15%
- **Test:** 15%

Class proportions were preserved across all three partitions.

---

## 2. Preprocessing

A leakage-safe preprocessing workflow was used.

### Numeric features
- Median imputation for missing values
- Standard scaling

### Categorical features
- Most-frequent imputation
- One-hot encoding
- Unknown categories ignored during transformation

The preprocessing transformer was fitted only on the training set.

After preprocessing:

- Train shape: **(7000, 42)**
- Validation shape: **(1500, 42)**
- Test shape: **(1500, 42)**
- No missing values remained in the processed training matrix.

---

## 3. Baseline Model Results

Four substantially different classification model families were evaluated on the validation set.

| Model | Accuracy | Precision | Recall | F1 | ROC-AUC | PR-AUC |
|---|---:|---:|---:|---:|---:|---:|
| Logistic Regression | 0.5373 | 0.0727 | 0.4153 | 0.1237 | 0.4715 | 0.0775 |
| Random Forest | 0.9213 | 0.0000 | 0.0000 | 0.0000 | 0.4475 | 0.0688 |
| XGBoost | 0.8187 | 0.0523 | 0.0763 | 0.0621 | 0.4496 | 0.0799 |
| LightGBM | 0.9087 | 0.0476 | 0.0085 | 0.0144 | 0.4549 | 0.0697 |

### Interpretation

The high accuracy of Random Forest and LightGBM was misleading because the dataset was strongly imbalanced.

For example, Random Forest predicted all validation customers as non-churn:

```text
[[1382    0]
 [ 118    0]]
```

This produced an accuracy of 92.13% but detected **zero churners**.

The more informative ranking metrics showed weak predictive performance:

- All ROC-AUC values were below 0.50.
- PR-AUC values remained close to the churn prevalence of 7.86%.
- No model demonstrated meaningful separation between churn and non-churn customers.

These results suggested that the main issue might be limited target signal rather than model selection alone.

---

## 4. Numeric Signal Diagnosis

Pearson correlations between numeric features and churn were examined.

The strongest observed correlations were:

| Feature | Correlation with Churn |
|---|---:|
| `sms_avg` | 0.022373 |
| `sms_avg_first3m` | 0.017468 |
| `complaints_total` | 0.016243 |
| `complaints_avg` | 0.016243 |
| `MonthlyCharges` | -0.010871 |
| `data_max` | -0.010202 |
| `call_max` | 0.009839 |
| `complaints_total_first3m` | 0.009432 |
| `complaints_change_3m` | -0.009101 |
| `call_avg_last3m` | 0.008561 |

### Interpretation

All correlations were extremely close to zero. The largest absolute correlation was approximately **0.022**, indicating almost no simple linear relationship between individual numeric variables and churn.

---

## 5. Categorical Signal Diagnosis

Churn rates were compared across customer categories.

### Gender

| Gender | Churn Rate | Customers |
|---|---:|---:|
| Male | 8.02% | 4,978 |
| Female | 7.71% | 5,022 |

### Region

| Region | Churn Rate | Customers |
|---|---:|---:|
| Urban | 8.12% | 3,287 |
| Suburban | 8.05% | 3,402 |
| Rural | 7.40% | 3,311 |

### Contract Type

| Contract Type | Churn Rate | Customers |
|---|---:|---:|
| Yearly | 8.82% | 2,006 |
| Prepaid | 7.67% | 4,041 |
| Monthly | 7.56% | 3,953 |

### Interpretation

Categorical churn-rate differences were also small. The largest observed difference among contract types was approximately **1.26 percentage points** between Yearly and Monthly customers. Gender and region showed even smaller separation.

---

## 6. Churn vs Non-Churn Feature Means

Selected behavioral features were compared directly between churners and non-churners.

| Feature | Non-Churn Mean | Churn Mean | Difference |
|---|---:|---:|---:|
| MonthlyCharges | 51.095953 | 50.227150 | -0.868803 |
| TenureMonths | 41.611331 | 41.606461 | -0.004869 |
| call_avg | 492.195052 | 494.827451 | 2.632400 |
| data_avg | 9.900067 | 9.875262 | -0.024805 |
| sms_avg | 24.498182 | 24.844360 | 0.346178 |
| complaints_total | 6.024419 | 6.171756 | 0.147336 |
| call_change_3m | -24.782906 | -23.046370 | 1.736537 |
| data_change_3m | -0.418742 | -0.415899 | 0.002843 |
| sms_change_3m | 0.026337 | -0.339271 | -0.365607 |
| complaints_change_3m | -0.012264 | -0.071247 | -0.058983 |
| call_trend_slope | -2.354599 | -2.248890 | 0.105708 |
| data_trend_slope | -0.038193 | -0.043472 | -0.005279 |
| complaints_trend_slope | -0.000765 | -0.001063 | -0.000299 |

### Interpretation

The churn and non-churn groups had very similar average values across most important behavioral variables. Tenure and average data usage were nearly identical; complaints differed only slightly; and recent behavioral changes and long-term trend slopes showed minimal separation.

---

## 7. Nonlinear Signal Diagnosis

Mutual Information was calculated to test whether nonlinear relationships might exist even when Pearson correlations were weak.

Top Mutual Information scores:

| Feature | Mutual Information |
|---|---:|
| `data_trend_slope` | 0.005488 |
| `sms_std` | 0.004306 |
| `MonthlyChargesInvalid` | 0.004257 |
| `call_std` | 0.003917 |
| `ContractType_Prepaid` | 0.003716 |
| `Region_Urban` | 0.003063 |
| `complaints_change_3m` | 0.002381 |
| `Region_Rural` | 0.002322 |
| `Gender_Female` | 0.002247 |
| `call_avg` | 0.001903 |
| `sms_avg` | 0.001393 |
| `call_max` | 0.001375 |
| `complaints_total_first3m` | 0.000954 |
| `complaints_max` | 0.000403 |
| `data_std` | 0.000378 |

### Interpretation

The highest Mutual Information score was only about **0.0055**. Most features had values extremely close to zero, suggesting that strong nonlinear relationships between the predictors and churn were also absent.

---

## 8. Overall Diagnosis

The evidence from multiple independent checks is consistent:

1. Linear and tree-based classifiers all performed near or below random ranking.
2. PR-AUC remained close to the churn prevalence baseline.
3. Numeric correlations with churn were extremely weak.
4. Categorical churn-rate differences were small.
5. Churn and non-churn groups had very similar feature means.
6. Mutual Information scores were also close to zero.

### Conclusion

The current dataset appears to contain **very weak predictive signal for churn using the available features**.

The weak model performance is therefore unlikely to be solved simply by trying additional classifiers, aggressive hyperparameter tuning, changing the probability threshold, or increasing model complexity.

---

## 9. Modeling Decision

At this stage, the appropriate decision is **not to continue hyperparameter tuning on this dataset**.

Models such as SVM and KNN could still be tested as supplementary benchmarks, but they are unlikely to materially change the conclusion because four substantially different model families have already shown similar limitations.

Instead, the project should proceed with a dataset that contains stronger and more realistic churn-related predictors.

Potentially useful churn predictors in a replacement dataset may include service tenure, contract duration, billing behavior, payment method, service subscriptions, support interactions, service quality indicators, price changes, customer engagement, usage decline, recent complaints, product/service changes, and account status changes.

---

## 10. Value of the Current Analysis

The work completed on this dataset should be retained rather than discarded.

It demonstrates an important applied machine-learning skill:

> **Recognizing when poor model performance is caused by weak data signal rather than blindly increasing model complexity.**

The project has already demonstrated:

- Data auditing
- Duplicate handling
- Missing-value analysis
- Data-quality flag construction
- Longitudinal feature engineering
- Behavioral trend engineering
- Leakage-safe preprocessing
- Class-imbalance awareness
- Stratified train/validation/test splitting
- Multi-model benchmarking
- Appropriate use of ROC-AUC and PR-AUC
- Correlation diagnostics
- Categorical target-rate analysis
- Mutual Information analysis
- Evidence-based model feasibility assessment

This diagnostic stage can therefore remain in the repository as a documented **model feasibility and weak-signal case study**.

---

## Final Recommendation
The current dataset should be retained as a documented weak-signal feasibility case because it demonstrates a complete diagnostic workflow, including data auditing, temporal feature engineering, imbalance-aware evaluation, multi-model benchmarking, correlation analysis, and nonlinear signal assessment.
Given the consistently weak predictive relationships observed across multiple model families and diagnostic tests, further tuning on this dataset is unlikely to produce a reliable churn classifier. Future development of the Telecom Churn Prediction & Retention Decision System should therefore use a dataset with stronger and more informative churn-related predictors.
The existing project framework can then be extended to model optimization, hyperparameter tuning, probability calibration, threshold optimization, SHAP-based explainability, customer risk segmentation, retention decision rules, business impact analysis, model persistence, and Streamlit deployment.
