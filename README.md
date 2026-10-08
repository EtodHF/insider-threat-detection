# 🛡️ Machine Learning-Based Insider Threat Detection Using Employee Behavioral Data

> **Data Science Capstone Project**
>
> **Author:** Ebubechukwu Ogbonna
>
> **Institution:** Hexagon Tech Hub, Ikeja GRA, Lagos

## 📌 Executive Overview

Insider threats present one of the most critical challenges in enterprise cybersecurity because perpetrators operate with legitimate credentials and permissions. Perimeter-based defenses (firewalls, standard identity verification) often fail to detect abnormal actions executed under authorized accounts.

This project implements and evaluates an end-to-end supervised machine learning pipeline designed to distinguish between benign employee activity and potential insider threats using corporate behavioral logs. Rather than optimizing purely for raw classification accuracy on an imbalanced dataset, the pipeline applies **data-minimization governance principles (NIST Privacy Framework)** and evaluates models using **Precision-Recall AUC, Recall, Precision, and F1-score**.

The resulting system is designed as an operational **decision-support prototype** for security operations center (SOC) analysts—prioritizing transparency and human-in-the-loop validation over automated employee surveillance or punitive decision-making.

## ⚖️ Ethical Governance \& Data Minimization

In compliance with the **NIST Privacy Framework** and principles of data minimization, security analytics must avoid ingesting sensitive personal information simply because it may contain statistical correlation.

* **Excluded Attributes:** Out of 21 raw candidates, sensitive personal, demographic, and medical attributes were intentionally excluded (`employee\_department`, `employee\_campus`, `employee\_position`, `employee\_origin\_country`, `has\_foreign\_citizenship`, `has\_criminal\_record`, `has\_medical\_history`, and `employee\_classification`).
* **Retained Feature Set (13 Numerical Behavioral Indicators):**

  * **File \& Exfiltration Metrics:** `total\_files\_burned`, `burned\_from\_other`
  * **Printing Behavior:** `total\_printed\_pages`, `num\_printed\_pages\_off\_hours`
  * **Physical Facility Access:** `num\_entries`, `num\_unique\_campus`, `late\_exit\_flag`, `entry\_during\_weekend`
  * **Travel \& Geopolitical Context:** `is\_abroad`, `trip\_day\_number`, `hostility\_country\_level`
  * **Organizational Tenancy:** `employee\_seniority\_years`, `is\_contractor`

## 📊 Dataset \& Exploratory Findings

* **Source:** *Insider Threat Dataset for Corporate Environments* (Kaggle)
* **Observations:** 118,614 records | 0 missing values | 0 duplicate records
* **Class Imbalance:**

  * **Benign Class (`0`):** 112,228 records (94.62%)
  * **Malicious Class (`1`):** 6,386 records (5.38%)
  * **Imbalance Ratio:** $\\approx 17.6 : 1$ (A baseline dummy classifier yields 94.62% accuracy, demonstrating that accuracy alone is an inadequate evaluation metric).

### Key Behavioral Indicators Observed

* **Digital Media Burning:** Malicious actors burned an average of **44.40 files** compared to **7.39 files** for benign users ($r = 0.423$). External source burning was nearly **92× higher** among malicious accounts ($r = 0.297$).
* **Off-Hours Physical \& Document Access:** Malicious personnel exhibited over **3× higher rates of weekend entry** ($0.1472$ vs. $0.0451$) and printed significantly more pages outside standard business hours ($4.13$ vs. $0.38$ pages).

## ⚙️ Methodology \& Modeling Pipeline

1. **Stratified Splitting:** 80/20 train-test split (`test\_size=0.2`, `random\_state=42`, stratified by target `is\_malicious`).
2. **Preprocessing Pipelines:**

   * `StandardScaler` applied to distance- and coefficient-based models (**Logistic Regression**, **KNN**).
   * Tree-based models (**Random Forest**) trained without scaling.
3. **Hyperparameter Tuning:**

   * Coarse search via `RandomizedSearchCV` followed by targeted refinement via `GridSearchCV`.
   * 5-Fold `StratifiedKFold` cross-validation optimized specifically on **F1-score**.

```
Raw Behavioral Data
        │
        ▼
Data Minimization (Filter sensitive PII/demographics)
        │
        ▼
Stratified Train/Test Split (80:20)
        │
   ┌────┴──────────────────────────┐
   ▼                               ▼
Scikit-Learn Pipeline           Random Forest
\[StandardScaler + Model]        \[Class Weighted]
   │                               │
   ├─ Logistic Regression          │
   └─ K-Nearest Neighbors          │
   │                               │
   └───────────────┬───────────────┘
                   ▼
  5-Fold Stratified Cross-Validation (F1-Optimization)
                   ▼
  Threshold \& Metric Evaluation (PR-AUC, ROC-AUC, F1, Recall)

```

## 📈 Model Performance \& Evaluation

All metrics evaluated on the unseen test set ($N = 23,723$ observations):

|Model|Accuracy|Precision|Recall|F1-Score|ROC-AUC|PR-AUC|
|-|-|-|-|-|-|-|
|**Logistic Regression** (Balanced)|87.46%|25.93%|71.65%|38.08%|0.874|0.581|
|**K-Nearest Neighbors (KNN)**|**96.83%**|**80.80%**|54.03%|**64.76%**|0.868|0.622|
|**Random Forest** (Balanced)|93.93%|46.11%|**75.18%**|57.16%|**0.921**|**0.686**|

### Confusion Matrix Breakdown

|Metric|Logistic Regression|K-Nearest Neighbors|Random Forest|
|-|-|-|-|
|**True Negatives (TN)**|19,832|**22,282**|21,324|
|**False Positives (FP)**|2,614|**164**|1,122|
|**False Negatives (FN)**|362|587|**317**|
|**True Positives (TP)**|915|690|**960**|

### Key Trade-Off Analysis

* **Random Forest** yielded the highest discriminatory capacity overall (**ROC-AUC: 0.921**, **PR-AUC: 0.686**) and minimized critical false negatives (detected 960 out of 1,277 malicious events, 75.18% recall). It is ideal where missing a threat is unacceptable.
* **KNN** achieved the highest precision (**80.80%**) and lowest false alarms (only 164 FPs), making it suited for bandwidth-constrained security teams that prioritize alert fidelity.

## 🔍 Feature Importance Analysis

Both MDI (Gini Impurity) and Permutation Importance confirmed document and media handling as dominant predictive drivers:

```
Random Forest Permutation Importance:
┌─────────────────────────────────┬───────────┐
│ Feature                         │ Importance│
├─────────────────────────────────┼───────────┤
│ total\_files\_burned              │  0.234390 │
│ total\_printed\_pages             │  0.081028 │
│ num\_unique\_campus               │  0.064606 │
│ num\_entries                     │  0.043756 │
│ num\_printed\_pages\_off\_hours     │  0.042295 │
│ burned\_from\_other               │  0.024614 │
│ hostility\_country\_level         │  0.010024 │
│ employee\_seniority\_years        │  0.009681 │
│ is\_contractor                   │  0.002747 │
│ entry\_during\_weekend            │  0.001279 │
│ trip\_day\_number                 │  0.001182 │
│ late\_exit\_flag                  │  0.000000 │
│ is\_abroad                       │ -0.002591 │
└─────────────────────────────────┴───────────┘

```

> \*Note: Feature importance represents statistical association within the trained model and does not imply direct behavioral causality.\*

## 📁 Repository Structure

```
insider-threat-detection/
├── notebooks/
│   ├── 01\_eda.ipynb                     # Data quality, distribution, and correlation analysis
│   └── 02\_modeling.ipynb                # Preprocessing, hyperparameter tuning, and model evaluation
├── models/
│   ├── logistic\_regression\_model.pkl    # Serialized tuned Logistic Regression pipeline
│   ├── knn\_model.pkl                    # Serialized tuned KNN pipeline
│   ├── random\_forest\_model.pkl          # Serialized tuned Random Forest model
│   └── feature\_names.pkl                # Exported list of final features
├── results/
│   ├── model\_evaluation\_results.csv     # Accuracy, precision, recall, F1, ROC-AUC, PR-AUC
│   ├── random\_forest\_feature\_importance.csv
│   └── permutation\_importance.csv
└── README.md                            # Project documentation

```

## 🚀 Quickstart \& Reproduction

### 1\. Clone the Repository

```
git clone https://github.com/<your-username>/insider-threat-detection.git
cd insider-threat-detection

```

### 2\. Set Up Virtual Environment

```
python -m venv venv
# On Linux/macOS:
source venv/bin/activate
# On Windows:
venv\\Scripts\\activate

```

### 3\. Run Notebooks

Launch Jupyter to inspect the analysis or re-run the experiments:

```
jupyter notebook notebooks/02\_modeling.ipynb

```

## 🛡️ IT Risk \& Governance Recommendations

1. **Human-in-the-Loop Decision Support:** Automated alerts should trigger internal review protocols rather than autonomous administrative actions.
2. **Dynamic Risk-Appetite Calibration:** Classification thresholds should be adjusted dynamically based on team capacity and organizational threat level (e.g., lower threshold during sensitive operational periods).
3. **Continuous Drift Monitoring:** Routine retraining is recommended to prevent degradation caused by evolving workforce work-from-home or travel patterns.

## 📚 References

* National Institute of Standards and Technology (NIST). *NIST Privacy Framework: A Tool for Improving Privacy Through Enterprise Risk Management, Version 1.0*. [https://www.nist.gov/privacy-framework](https://www.nist.gov/privacy-framework)
* NIST Computer Security Resource Center (CSRC) Glossary: *Insider Threat*, *User Activity Monitoring*, *Minimization*.
* Uzaki, A. *Insider Threat Dataset for Corporate Environments*. Kaggle.

