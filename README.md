# Loan Application Refusal Prediction & Credit Decision Dashboard

NeoStats Data Science Hackathon submission.

## Problem

Predict, at application time, whether a loan application will be **Refused** by the lender, and present results in a Power BI dashboard.

## Target

`REFUSED` built from `NAME_CONTRACT_STATUS`:

- Approved -> 0
- Refused -> 1
- Canceled, Unused offer -> removed (the customer, not the lender, decided)

Actual refusal rate in cleaned data: **21.90%** (290,678 refused of 1,327,459 after removing Canceled/Unused)

## Project structure

```
refusal_prediction/
├── data/            raw files (not committed)
├── notebooks/       01_data_preparation, 02_eda, 03_modelling_validation
├── outputs/         tables exported for the dashboard
├── models/          final_model.joblib
├── dashboard/       refusal_dashboard.pbix
├── documents/       project_presentation.pdf
├── requirements.txt
└── README.md
```

## How to run

1. Download the data from https://github.com/neoYogesh/Neo_DS_Hackathon/releases/tag/data-v1 and place `previous_application.csv`, `application_train.csv` and `HomeCredit_columns_description.csv` in `data/`.
2. `pip install -r requirements.txt`
3. Run notebooks in order: `01_data_preparation` -> `02_eda` -> `03_modelling_validation`.
4. Open `dashboard/refusal_dashboard.pbix` in Power BI Desktop (refresh the CSV paths if needed).

## Data used

Only `previous_application.csv` (main table), `application_train.csv` (customer attributes) and the column dictionary. Other files in the raw data folder are not used.

## Leakage columns removed

14 columns were dropped from `previous_application.csv`. Each was tested empirically (missingness compared between approved and refused rows, plus targeted cross-checks) rather than assumed.

| Column | Reason |
|---|---|
| `NAME_CONTRACT_STATUS` | This *is* the target. |
| `CODE_REJECT_REASON` | Crosstab showed `XAP` on all 1,036,773 approved rows and real reasons only on refused rows — it encodes the outcome. |
| `AMT_CREDIT` | Final amount granted by the lender (post-decision). Only 18.99% of approved vs 41.59% of refused rows have it equal to the request, so it is an approval artefact. We keep `AMT_APPLICATION` (the amount the customer asked for). |
| `AMT_ANNUITY` | Missing on 14.1% of refused vs 0.0% of approved — set during approval. |
| `CNT_PAYMENT` | Same pattern (14.1% vs 0.0%). |
| `RATE_DOWN_PAYMENT` | Rate normalised on the granted credit; missing 69.9% vs 36.1%. |
| `RATE_INTEREST_PRIMARY` | Lender-set interest rate; ~100% missing on refused rows. |
| `RATE_INTEREST_PRIVILEGED` | Same. |
| `NFLAG_INSURED_ON_APPROVAL` | Insurance flag set after approval; 100% missing on refused. |
| `DAYS_FIRST_DRAWING` | Disbursement date — exists only if the credit was drawn. |
| `DAYS_FIRST_DUE` | Same. |
| `DAYS_LAST_DUE_1ST_VERSION` | Same. |
| `DAYS_LAST_DUE` | Same. |
| `DAYS_TERMINATION` | Same. |

`AMT_DOWN_PAYMENT` was **kept** despite a missingness gap (36.1% vs 69.9%) because it is an application-time term the customer proposes. LightGBM handles the missing values natively; this asymmetry is recorded as a limitation below.

After merging, all 120 `application_train` columns were renamed with a `CUST_` prefix so application-level and customer-level fields cannot be confused (`CUST_AMT_CREDIT` is the customer's *current* application credit, not the previous application's).

## Split strategy

- OOT: `DAYS_DECISION > -365` (306,791 rows, refusal rate 37.26%)
- Development: `DAYS_DECISION <= -365` (1,020,668 rows, refusal rate 17.28%), split 70/15/15 stratified → train 714,467 / val 153,100 / test 153,101, all at 17.28%
- Preprocessing fitted on train only; random seed = 42
- The same `SK_ID_PREV` never appears in two splits.

**Important finding:** the refusal rate nearly doubled from 17.35% (older data) to 37.39% (most recent 12 months). Drift analysis showed this is a **policy change, not a population change** — the largest feature mean shift was only 0.32 standard deviations, and the largest missingness gap was 2.73 percentage points. The OOT year is genuinely harder, and it is the honest test of the model.

## Class imbalance approach

Refused rows are 21.90% of the data (1 : 3.57 in train). Two measures, both fitted on train only:

- **`scale_pos_weight = 4.79`** passed to LightGBM (the approved:refused ratio in train), so the loss treats a missed refusal as roughly as costly as a false alarm.
- **Threshold tuned on validation**, not left at the default 0.5 (see below).

Accuracy was rejected as a metric: a model that always answers "Approved" scores 82.72% accuracy while catching 0% of refusals. We report **ROC-AUC** and **average precision** instead.

## Models, KPI comparison, final choice and threshold

Models compared, all on the validation split, seed 42:

| Model | Val ROC-AUC | Val avg precision | Note |
|---|---|---|---|
| Dummy (prior) | 0.5000 | 0.1728 | Equals the base rate — the bar to clear |
| **LightGBM** | **0.8309** | **0.6268** | **Chosen** |

LightGBM configuration: 2000 max trees, early stopping on validation AUC (patience 50), learning rate 0.05, 31 leaves, row/column subsampling 0.9, `scale_pos_weight = 4.79`. Early stopping fired at **10 of 2000** trees — the categorical product features carry the signal almost immediately. Final model is 23.5 KB.

**Threshold:** chosen on validation by maximising F1 → **0.35**. This is the single most important operational decision. At the default 0.5 the model has precision 1.0 but recall 0.12 — it would miss 88% of refusals. At 0.35 it achieves precision 0.59 / recall 0.55. The threshold is stored in the notebook and in the `PRED_REFUSED` column of the predictions table; it is not a model parameter.

**Feature importance (gain):** driven overwhelmingly by application/product attributes, not customer identity. Top features: `NAME_CLIENT_TYPE` 25.1%, `PRODUCT_COMBINATION` 20.3%, `NAME_PORTFOLIO` 16.3%, `NAME_GOODS_CATEGORY` 10.2%, `NAME_PRODUCT_TYPE` 9.0%, `FLAG_LAST_APPL_PER_CONTRACT` 5.3%. The top six application-level features account for ~86% of total gain; the best customer feature, `CUST_EXT_SOURCE_3`, contributes 1.6%. In plain terms: *what product you apply for and whether you are a repeater* predicts refusal far better than who you are.

## Test vs OOT results

| Split | Rows | Refusal rate | ROC-AUC | Avg precision | Precision | Recall | F1 |
|---|---|---|---|---|---|---|---|
| val | 153,100 | 17.28% | 0.8309 | 0.6268 | 0.59 | 0.55 | 0.568 |
| test | 153,101 | 17.28% | 0.8281 | 0.6204 | 0.59 | 0.54 | 0.562 |
| OOT (last 12 months) | 306,791 | 37.26% | 0.7982 | 0.7488 | 0.71 | 0.55 | 0.616 |

Reading these honestly:

- **Test ≈ val (0.8281 vs 0.8309)** — no overfitting; the split strategy is sound.
- **OOT AUC falls to 0.7982** — expected, the refusal policy shifted and the year is genuinely harder.
- **OOT average precision *rises* to 0.7488** — not a contradiction. Average precision is sensitive to the base rate, which doubled. AUC measures ranking quality across all thresholds and is the better cross-period comparison; AP should always be quoted next to the refusal rate for its split.
- **Recall is stable at ~0.55 across all three splits** — the model's ability to actually find refusals does not degrade out of sample.

## Dashboard / data model

Built in Power BI Desktop from the CSVs in `outputs/`. **Spec lives in `dashboard/README_DASHBOARD.md`** — it lists every table to load, every DAX measure, and the visual-by-visual layout for four pages (Overview, Who gets refused, Amount effects, Model quality).

Data model: nine flat tables loaded independently, **no relationships** — each visual reads one already-aggregated table, so nothing can silently cross-join. Measures are simple (`Refusal Rate %`, `Applications`, `Refused Count`, `Avg Predicted Prob`, `Precision at 0.35`).

Key dashboard story, all from executed outputs: refusal rate rose 17.35% → 37.39% over the most recent 12 months; the model is driven by product/application attributes (top 6 ≈ 86% of gain), not customer identity; test AUC 0.8281 and OOT AUC 0.7982 at threshold 0.35.

## Limitations and improvements

Honest constraints, in rough order of importance:

1. **The same customer can appear in both train and test.** There are 338,857 unique customers behind 1.33M applications (~3.9 applications each), and the split is stratified by row, not grouped by customer. A customer in both splits lets the model partly memorise them, so test metrics are slightly optimistic. A group-aware split (`GroupShuffleSplit` on `SK_ID_CURR`) is the correct fix and was not done here.
2. **Customer attributes are observed at the *current* application, which is later in time than the previous decision being predicted.** This is a mild time inconsistency in the source data — `application_train` describes the customer's application at time T, while `previous_application` rows describe decisions before T. Real deployment would only have features available at decision time.
3. **`AMT_DOWN_PAYMENT` recording depends on the outcome** (missing on 69.9% of refused vs 36.1% of approved rows). It is kept as an application-time term, so its *missingness* carries some signal about the decision. LightGBM handles the missing values, but this is a residual leakage risk worth naming.
4. **The 17% → 37% refusal-rate jump is a policy change the model cannot explain.** No feature in the data moved enough (max shift 0.32 SD) to account for it. AUC drops on OOT for this reason, not because the model is broken. If the lender's policy shifts again, expect the same.
5. **`DAYS_DECISION` is used only to define the split, never as a feature** — deliberately, so the model learns applicant/application attributes rather than "recent era = more refusals". A time-feature variant is a possible experiment but would weaken the OOT test.
6. **`MONTHS_AGO` in the trend chart is relative, not calendar.** `DAYS_DECISION` is measured against each customer's own current application, so there are no absolute dates. Month 12 sits on the OOT boundary and contains a few development rows.

Improvements, in the order they would pay off:

- Group-aware split by `SK_ID_CURR` (fixes limitation 1).
- Tune the threshold per business cost rather than by F1 — if a missed refusal costs more than a false alarm, push recall up and accept lower precision.
- Add the unused Home Credit tables (bureau, installments, POS/CASH) as per-customer history aggregates; they are not used here.
- Calibrate probabilities (isotonic on validation) if the scores will be used as probabilities rather than for ranking.
- Try the 46 mostly-missing `CUST_*_AVG/_MODE/_MEDI` building-area features explicitly; LightGBM will ignore useless ones, but they were kept and may carry a little housing signal.
