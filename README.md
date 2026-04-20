# NSL-KDD Network Intrusion Detection — CAP 4770 / 6771 Final Project

Binary classification of network connections as **Normal (0)** or **Anomaly/Attack (1)** using the NSL-KDD dataset, with data sourced directly from a **MySQL database** and a full machine learning pipeline built in Jupyter.

---

## Project Overview

| Item | Detail |
|---|---|
| Course | CAP 4770 / 6771 — Data Mining |
| Dataset | NSL-KDD (Canadian Institute for Cybersecurity) |
| Task | Binary classification: Normal vs Anomaly |
| Database | MySQL — `nsl_kdd` database |
| Best model | Random Forest — F1: **0.9765**, AUC-ROC: **0.9980**, Accuracy: **99.39%** |
| Train records | 10,264 (9,276 normal · 988 anomaly) |
| Test records | 2,803 (2,444 normal · 359 anomaly) |

---

## Repository Contents

```
NSL_KDD-CAP4770-Final-Project/
│
├── cap4770_6771_mysql.ipynb              # MySQL connection reference notebook
│                                         # Establishes the connection pattern used
│                                         # throughout the project (use_pure=True)
│
├── NSL_KDD_PreProcess.ipynb              # Step 1 — Feature mapping
│                                         # Maps raw 42-col NSL-KDD CSV →
│                                         # 13-feature common schema CSV
│
├── Cleaning.ipynb                        # Step 2 — Data cleaning pipeline
│                                         # IQR outlier capping, normalisation,
│                                         # duplicate removal, schema validation
│
├── NSL_KDD_Classification_MySQL (1).ipynb  # Classification v1
│                                            # Uses imbalanced-learn SMOTE
│
├── NSL_KDD_Classification_MySQL (2).ipynb  # Classification v2 (FINAL)
│                                            # Uses sklearn.utils.resample
│                                            # No imbalanced-learn dependency
│
└── NSL_KDD_Paper_Final.docx              # Written report — academic paper format
```

---

## Pipeline

The project follows a five-stage pipeline:

```
MySQL (nsl_kdd)
    └─ pd.read_sql()
        └─ NSL_KDD_PreProcess.ipynb      ← map 42 raw cols → 13 common features
            └─ Cleaning.ipynb            ← clean, cap outliers, validate schema
                └─ Classification (2)    ← encode, log1p, scale, oversample, train
                    └─ Evaluation        ← metrics, confusion matrix, ROC, importance
```

---

## MySQL Setup

The two cleaned tables must exist in a local MySQL database named `nsl_kdd`:

```sql
CREATE DATABASE IF NOT EXISTS nsl_kdd;
USE nsl_kdd;
-- then import:
-- nsl_kdd_common_train_clean
-- nsl_kdd_common_test_clean
```

The connection used throughout all notebooks:

```python
import mysql.connector as sql

conn = sql.connect(
    host     = 'localhost',
    user     = 'root',
    password = 'bob123',
    database = 'nsl_kdd',
    use_pure = True          # required — avoids C-extension database lookup bug
)
```

> **Note:** Update `password` to match your local MySQL root password.  
> If your server runs on a non-default port, add `port=XXXX`.

---

## Installation

```bash
# MySQL connector (via conda — matches course setup)
%conda install -c anaconda mysql-connector-python -y

# Python packages
pip install scikit-learn pandas numpy matplotlib seaborn
```

> The final notebook (`_MySQL (2).ipynb`) uses only `sklearn.utils.resample` for
> oversampling — **no imbalanced-learn required**. If you want to run `_MySQL (1).ipynb`
> you will need `scikit-learn==1.4.2` and `imbalanced-learn==0.12.3` pinned together
> (sklearn 1.8+ broke the imbalanced-learn 0.14.x API).

---

## Run Order

Run the notebooks in this order:

1. **`NSL_KDD_PreProcess.ipynb`** — point `df = pd.read_csv(...)` at your raw NSL-KDD CSV files and run to produce the 13-feature CSVs.
2. **`Cleaning.ipynb`** — update `TRAIN_IN` / `TEST_IN` paths to the CSVs from step 1. Outputs cleaned CSVs ready for MySQL import.
3. Import the cleaned CSVs into MySQL (`nsl_kdd` database) as `nsl_kdd_common_train_clean` and `nsl_kdd_common_test_clean`.
4. **`NSL_KDD_Classification_MySQL (2).ipynb`** — runs the full ML pipeline end-to-end from MySQL.

---

## Feature Schema

The pipeline maps the raw NSL-KDD 42-column format to 13 common features. Five are dropped before modelling (constant zero in NSL-KDD):

| Feature | Source mapping | Used? |
|---|---|---|
| `flow_duration` | `duration` | ✅ log1p → scaled |
| `protocol` | `protocol_type` | ✅ label encoded |
| `service_state` | `service + "_" + flag` | ✅ label encoded |
| `fwd_bytes` | `src_bytes` | ✅ log1p → scaled |
| `bwd_bytes` | `dst_bytes` | ✅ log1p → scaled |
| `packet_rate` | `count / duration` | ✅ log1p → scaled |
| `byte_rate` | `(src+dst) / duration` | ✅ log1p → scaled |
| `avg_packet_size` | `(src+dst) / count` | ✅ log1p → scaled |
| `fwd_packet_count` | not in NSL-KDD | ❌ constant 0, dropped |
| `bwd_packet_count` | not in NSL-KDD | ❌ constant 0, dropped |
| `syn_flag_count` | not in NSL-KDD | ❌ constant 0, dropped |
| `rst_flag_count` | not in NSL-KDD | ❌ constant 0, dropped |
| `ack_flag_count` | not in NSL-KDD | ❌ constant 0, dropped |

---

## Models & Results

Six classifiers evaluated on the held-out test set (n = 2,803) with random oversampling applied to the training set:

| Model | Accuracy | Precision | Recall | F1-Score | AUC-ROC |
|---|---|---|---|---|---|
| Logistic Regression | 0.8980 | 0.5628 | 0.9109 | 0.6957 | 0.9364 |
| Naive Bayes | 0.9026 | 0.5760 | 0.9081 | 0.7049 | 0.9466 |
| KNN (k=5) | 0.9850 | 0.9054 | 0.9861 | 0.9440 | 0.9948 |
| Decision Tree | 0.9918 | 0.9590 | 0.9777 | 0.9683 | 0.9871 |
| **Random Forest** | **0.9939** | **0.9698** | **0.9833** | **0.9765** | **0.9980** |
| SVM (RBF) | 0.9818 | 0.8869 | 0.9833 | 0.9326 | 0.9948 |

**Random Forest confusion matrix (test set):**

```
                Predicted Normal   Predicted Anomaly
True Normal          2,433               11
True Anomaly             6              353
```

Only 17 total misclassifications out of 2,803 test records.

---

## Top Feature Importances (Random Forest — Gini)

| Feature | Importance |
|---|---|
| `log_bwd_bytes` | 32.0% |
| `log_fwd_bytes` | 25.7% |
| `log_avg_packet_size` | 18.9% |
| `service_state` | 12.2% |
| `protocol` | 4.9% |
| `log_byte_rate` | 2.2% |
| `log_packet_rate` | 2.1% |
| `log_flow_duration` | 2.0% |

The top 4 features account for **76.6%** of predictive power. Byte volume is the primary signal for attack detection.

---

## Classification Notebook Walkthrough (`_MySQL (2).ipynb`)

| Section | What it does |
|---|---|
| 1. Environment | `!python --version` |
| 2. Install | `%conda` + `pip` for all dependencies |
| 3. MySQL connection | `sql.connect(...)` → `SHOW TABLES` |
| 4. Load data | `pd.read_sql()` for both tables, `conn.close()` |
| 5. Data overview | dtypes, missing values, `describe()` |
| 6. EDA | Class distribution, protocol breakdown, feature distributions, correlation heatmap, class means |
| 7. Preprocessing | Zero-var drop → label encode → log1p → StandardScaler → resample |
| 8. Training & CV | All 6 models, stratified 5-fold CV bar chart |
| 9. Evaluation | Metrics table, bar chart, 6 confusion matrices, ROC curves, classification reports |
| 10. No-oversampling comparison | Side-by-side F1 chart |
| 11. Feature importance | Random Forest Gini horizontal bar chart |
| 12. Summary | Final results + pipeline recap |

---

## Common Issues

**`ProgrammingError: Unknown database 'nsl_kdd'`**  
The database exists but the C extension cannot find it. Add `use_pure=True` to your `sql.connect()` call.

**`ImportError: cannot import name '_is_pandas_df'`**  
You have `scikit-learn 1.8+` which broke `imbalanced-learn 0.14.x`. Use the final notebook (`_MySQL (2).ipynb`) which replaces SMOTE with `sklearn.utils.resample` — no version pinning needed.

**`Unknown database`** even with `use_pure=True`  
Run `SHOW DATABASES;` in MySQL Workbench to confirm the exact database name, then re-check the `database=` argument.
