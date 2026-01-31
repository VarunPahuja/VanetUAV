# Modelling Pipeline Summary – CIC-IDS2017

This document summarizes the complete modelling workflow carried out on the CIC-IDS2017 dataset, including data preparation, exploratory analysis, feature selection, leakage detection, and baseline modelling. The goal was to build a **realistic and reproducible intrusion detection baseline**, avoiding common pitfalls such as data leakage and overly optimistic evaluation.

---

## 1. Dataset Preparation

- The CIC-IDS2017 dataset consists of 8 CSV files, each corresponding to a specific day and attack scenario.
- All CSV files were merged into a single dataset after:
  - Normalizing column names
  - Ensuring consistent feature sets
- Metadata columns were added:
  - `source_file` (origin CSV)
  - `day`
  - `attack_group`

The final merged dataset contained approximately **2.83 million rows and 79 features**.

---

## 2. Stratified 10% Subsampling

To enable faster experimentation while preserving dataset characteristics:
- A **10% stratified sample** was created
- The class distribution (Benign vs Attack) was preserved
- This reduced dataset was saved as:
  - `cicids_10pct_stratified.csv`

This dataset was used for all subsequent analysis and modelling.

---

## 3. Exploratory Data Analysis (EDA)

Key EDA findings:
- Strong class imbalance (~80% Benign, ~20% Attack)
- Presence of:
  - Near-zero variance features (e.g., protocol flag counters)
  - Infinite and NaN values in rate-based features
- Many features exhibited extreme skew and heavy tails

These observations motivated further feature pruning and leakage checks.

---

## 4. PCA-Based Feature Analysis

Principal Component Analysis (PCA) was performed on the 10% dataset after:
- Removing non-numeric columns
- Standardizing features
- Handling NaN and infinite values

Results:
- ~21 principal components captured 90% of the variance
- ~25 principal components captured 95% of the variance
- Feature importance was estimated by summing absolute loadings across top PCs

This step identified features that dominated variance but **did not automatically imply modelling suitability**, as PCA does not detect label leakage.

---

## 5. Initial Modelling and Leakage Detection

### 5.1 Initial Logistic Regression (Random Split)

A Logistic Regression baseline was trained using a **random train–test split**, which resulted in:
- Accuracy ≈ 1.00
- AUC ≈ 1.00

This indicated **severe data leakage**.

### 5.2 Root Cause Analysis

Multiple leakage sources were identified:
1. **Direct label leakage**
   - A derived binary label (`label_binary`) was accidentally included in the feature set
2. **Feature-level leakage**
   - Protocol flag counters (e.g., SYN, ACK, FIN)
   - Flow-final statistics (e.g., Flow Bytes/s, Idle Mean)
3. **Scenario / temporal leakage**
   - Random splits caused flows from the same attack instances to appear in both train and test sets

All leaky features were explicitly removed.

---

## 6. Clean Baseline with Random Split

After removing:
- Label columns
- Protocol flag counters
- Flow-final and bulk aggregation features

Logistic Regression was retrained using a random split:

- Accuracy: **0.94**
- AUC: **0.98**
- Attack recall: **0.84**

Although realistic at first glance, this was still considered **optimistic** due to scenario leakage.

---

## 7. Scenario-Based (File-Level) Evaluation

To simulate real-world deployment:
- Training and testing were split **by source file**, not by random rows
- Entire days / attack scenarios were held out for testing

### Results (Logistic Regression, file-based split):

- Accuracy: **0.70**
- Benign recall: **1.00**
- Attack recall: **0.28**
- Attack F1-score: **0.44**

---

## 8. Key Findings

1. Random train–test splits **significantly overestimate IDS performance** on CIC-IDS2017
2. Logistic Regression struggles to generalize to **unseen attack scenarios**
3. Attack detection is substantially harder than benign traffic classification
4. Scenario-based evaluation is essential for realistic IDS assessment

---

## 9. Final Baseline Summary

| Model | Split Strategy | Accuracy | Attack Recall | AUC |
|------|---------------|----------|---------------|-----|
| Logistic Regression | Random | 0.94 | 0.84 | 0.98 |
| Logistic Regression | File-based | **0.70** | **0.28** | — |

The file-based result is treated as the **true baseline**.

---

## 10. Next Steps

- Train non-linear models (e.g., Random Forest) under file-based evaluation
- Compare full feature set vs PCA-selected features
- Extend the setup to **federated learning**, where each source file acts as a client

---

## Conclusion

This workflow demonstrates the importance of:
- Rigorous leakage detection
- Proper evaluation protocols
- Scenario-aware validation

The resulting baseline provides a **credible and reproducible foundation** for further IDS and federated learning research.

