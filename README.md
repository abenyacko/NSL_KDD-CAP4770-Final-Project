NSL-KDD Network Intrusion Detection — CAP 4770 / 6771 Final Project
Binary classification of network connections as Normal (0) or Anomaly/Attack (1) using the NSL-KDD dataset, with data sourced directly from a MySQL database and a full machine learning pipeline built in Jupyter.

Project Overview
ItemDetailCourseCAP 4770 / 6771 — Data MiningDatasetNSL-KDD (Canadian Institute for Cybersecurity)TaskBinary classification: Normal vs AnomalyDatabaseMySQL — nsl_kdd databaseBest modelRandom Forest — F1: 0.9765, AUC-ROC: 0.9980, Accuracy: 99.39%Train records10,264 (9,276 normal · 988 anomaly)Test records2,803 (2,444 normal · 359 anomaly)

Repository Contents
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

Pipeline
The project follows a five-stage pipeline:
MySQL (nsl_kdd)
    └─ pd.read_sql()
        └─ NSL_KDD_PreProcess.ipynb      ← map 42 raw cols → 13 common features
            └─ Cleaning.ipynb            ← clean, cap outliers, validate schema
                └─ Classification (2)    ← encode, log1p, scale, oversample, train
                    └─ Evaluation        ← metrics, confusion matrix, ROC, importance

MySQL Setup
The two cleaned tables must exist in a local MySQL database named nsl_kdd:
sqlCREATE DATABASE IF NOT EXISTS nsl_kdd;
USE nsl_kdd;
-- then import:
-- nsl_kdd_common_train_clean
-- nsl_kdd_common_test_clean
The connection used throughout all notebooks:
pythonimport mysql.connector as sql

conn = sql.connect(
    host     = 'localhost',
    user     = 'root',
    password = 'bob123',
    database = 'nsl_kdd',
    use_pure = True          # required — avoids C-extension database lookup bug
)

Note: Update password to match your local MySQL root password.
If your server runs on a non-default port, add port=XXXX.


Installation
bash# MySQL connector (via conda — matches course setup)
%conda install -c anaconda mysql-connector-python -y

# Python packages
pip install scikit-learn pandas numpy matplotlib seaborn

The final notebook (_MySQL (2).ipynb) uses only sklearn.utils.resample for
oversampling — no imbalanced-learn required. If you want to run _MySQL (1).ipynb
you will need scikit-learn==1.4.2 and imbalanced-learn==0.12.3 pinned together
(sklearn 1.8+ broke the imbalanced-learn 0.14.x API).


Run Order
Run the notebooks in this order:

NSL_KDD_PreProcess.ipynb — point df = pd.read_csv(...) at your raw NSL-KDD CSV files and run to produce the 13-feature CSVs.
Cleaning.ipynb — update TRAIN_IN / TEST_IN paths to the CSVs from step 1. Outputs cleaned CSVs ready for MySQL import.
Import the cleaned CSVs into MySQL (nsl_kdd database) as nsl_kdd_common_train_clean and nsl_kdd_common_test_clean.
NSL_KDD_Classification_MySQL (2).ipynb — runs the full ML pipeline end-to-end from MySQL.


Feature Schema
The pipeline maps the raw NSL-KDD 42-column format to 13 common features. Five are dropped before modelling (constant zero in NSL-KDD):
FeatureSource mappingUsed?flow_durationduration✅ log1p → scaledprotocolprotocol_type✅ label encodedservice_stateservice + "_" + flag✅ label encodedfwd_bytessrc_bytes✅ log1p → scaledbwd_bytesdst_bytes✅ log1p → scaledpacket_ratecount / duration✅ log1p → scaledbyte_rate(src+dst) / duration✅ log1p → scaledavg_packet_size(src+dst) / count✅ log1p → scaledfwd_packet_countnot in NSL-KDD❌ constant 0, droppedbwd_packet_countnot in NSL-KDD❌ constant 0, droppedsyn_flag_countnot in NSL-KDD❌ constant 0, droppedrst_flag_countnot in NSL-KDD❌ constant 0, droppedack_flag_countnot in NSL-KDD❌ constant 0, dropped

Models & Results
Six classifiers evaluated on the held-out test set (n = 2,803) with random oversampling applied to the training set:
ModelAccuracyPrecisionRecallF1-ScoreAUC-ROCLogistic Regression0.89800.56280.91090.69570.9364Naive Bayes0.90260.57600.90810.70490.9466KNN (k=5)0.98500.90540.98610.94400.9948Decision Tree0.99180.95900.97770.96830.9871Random Forest0.99390.96980.98330.97650.9980SVM (RBF)0.98180.88690.98330.93260.9948
Random Forest confusion matrix (test set):
                Predicted Normal   Predicted Anomaly
True Normal          2,433               11
True Anomaly             6              353
Only 17 total misclassifications out of 2,803 test records.

Top Feature Importances (Random Forest — Gini)
FeatureImportancelog_bwd_bytes32.0%log_fwd_bytes25.7%log_avg_packet_size18.9%service_state12.2%protocol4.9%log_byte_rate2.2%log_packet_rate2.1%log_flow_duration2.0%
The top 4 features account for 76.6% of predictive power. Byte volume is the primary signal for attack detection.

Classification Notebook Walkthrough (_MySQL (2).ipynb)
SectionWhat it does1. Environment!python --version2. Install%conda + pip for all dependencies3. MySQL connectionsql.connect(...) → SHOW TABLES4. Load datapd.read_sql() for both tables, conn.close()5. Data overviewdtypes, missing values, describe()6. EDAClass distribution, protocol breakdown, feature distributions, correlation heatmap, class means7. PreprocessingZero-var drop → label encode → log1p → StandardScaler → resample8. Training & CVAll 6 models, stratified 5-fold CV bar chart9. EvaluationMetrics table, bar chart, 6 confusion matrices, ROC curves, classification reports10. No-oversampling comparisonSide-by-side F1 chart11. Feature importanceRandom Forest Gini horizontal bar chart12. SummaryFinal results + pipeline recap

Common Issues
ProgrammingError: Unknown database 'nsl_kdd'
The database exists but the C extension cannot find it. Add use_pure=True to your sql.connect() call.
ImportError: cannot import name '_is_pandas_df'
You have scikit-learn 1.8+ which broke imbalanced-learn 0.14.x. Use the final notebook (_MySQL (2).ipynb) which replaces SMOTE with sklearn.utils.resample — no version pinning needed.
Unknown database even with use_pure=True
Run SHOW DATABASES; in MySQL Workbench to confirm the exact database name, then re-check the database= argument.
