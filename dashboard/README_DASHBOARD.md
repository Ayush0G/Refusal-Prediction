# Power BI dashboard spec - refusal_prediction

Build this in **Power BI Desktop** (free). Import the CSVs from `outputs/` — no database needed.
All numbers below come from the executed notebooks; do not invent others.

## 1. Get data

Home menu → **Get data → Text/CSV** → pick each file, **Load** (do not Transform unless stated).

| File | Load as | Role |
|---|---|
| `outputs/stage1_modeling.parquet` | Table `data` | Main table (1,327,459 rows). If Power BI cannot read `.parquet`, run the converter cell at the bottom of this file first. |
| `outputs/dashboard_predictions.csv` | Table `pred` | Model scores for test + OOT (459,892 rows) |
| `outputs/dashboard_time_trend.csv` | Table `trend` | Refusal % by month |
| `outputs/dashboard_model_metrics.csv` | Table `metrics` | val / test / OOT scores |
| `outputs/dashboard_feature_importance.csv` | Table `importance` | Top features by gain |
| `outputs/dashboard_refusal_by_category.csv` | Table `cat_app` | Refusal % by application category |
| `outputs/dashboard_refusal_by_customer_category.csv` | Table `cat_cust` | Refusal % by customer category |
| `outputs/dashboard_refusal_by_band.csv` | Table `bands` | Refusal % by numeric decile |
| `outputs/dashboard_drift.csv` | Table `drift` | dev vs OOT drift |

**No relationships needed** — every table is already aggregated or is a standalone slice. If Power BI auto-creates relationships, delete them (Model view → delete the lines) so each visual reads one table.

## 2. DAX measures

Modeling → **New measure**. Create these five:

```dax
Refusal Rate % = AVERAGE ( data[REFUSED] )
```
```dax
Applications = COUNTROWS ( data )
```
```dax
Refused Count = CALCULATE ( COUNTROWS ( data ), data[REFUSED] = 1 )
```
```dax
Avg Predicted Prob = AVERAGE ( pred[PROB_REFUSED] )
```
```dax
Precision at 0.35 =
VAR Confusion =
    CALCULATE (
        COUNTROWS ( pred ),
        pred[PRED_REFUSED] = 1 && pred[REFUSED] = 1
    )
VAR PredictedPos =
    CALCULATE ( COUNTROWS ( pred ), pred[PRED_REFUSED] = 1 )
RETURN
    DIVIDE ( Confusion, PredictedPos )
```

Put `Refusal Rate %` in the **card** visuals and set its format to **Percentage → 2 decimals** (Measure tools tab).

## 3. Pages and visuals

Four pages. For each: click the visual, set the **Fields** as shown.

### Page 1 - Overview
| Visual | Field(s) | Notes |
|---|---|---|
| Card | `metrics` → `roc_auc`, filter `split = test` | Title: "Model AUC (test)" |
| Card | `Refusal Rate %` (from `data`) | Title: "Refusal rate (all)" |
| Card | `metrics` → `avg_precision`, filter `split = OOT (last 12 months)` | Title: "Avg precision (OOT)" |
| Line chart | X = `trend[MONTHS_AGO]`, Y = `trend[refusal_pct]` | **Sort X descending** so recent is on the right. Title: "Refusal rate over time" |
| Clustered bar | X = `importance[feature]`, Y = `importance[gain_pct]`, Top 10 by gain_pct | Title: "What drives refusal (top 10)" |

### Page 2 - Who gets refused
| Visual | Field(s) |
|---|---|
| Clustered bar | X = `cat_app[category]`, Y = `cat_app[refusal_rate]`, filter `feature = NAME_CLIENT_TYPE` |
| Clustered bar | X = `cat_app[category]`, Y = `cat_app[refusal_rate]`, filter `feature = CHANNEL_TYPE` |
| Clustered bar | X = `cat_cust[category]`, Y = `cat_cust[refusal_rate]`, filter `feature = CUST_NAME_EDUCATION_TYPE` |
| Matrix | Rows = `cat_cust[feature]`, Values = `refusal_rate`, filter `feature` in (CUST_CODE_GENDER, CUST_NAME_INCOME_TYPE) |

### Page 3 - Amount effects
| Visual | Field(s) |
|---|---|
| Line chart | X = `bands[band_label]`, Y = `bands[refusal_rate]`, filter `feature = AMT_APPLICATION` | Shows the U-shape |
| Line chart | X = `bands[band_label]`, Y = `bands[refusal_rate]`, filter `feature = CUST_EXT_SOURCE_2` |
| Line chart | X = `bands[band_label]`, Y = `bands[refusal_rate]`, filter `feature = CUST_AMT_INCOME_TOTAL` |

### Page 4 - Model quality
| Visual | Field(s) |
|---|---|
| Matrix | Rows = `metrics[split]`, Values = `roc_auc`, `avg_precision`, `precision`, `recall`, `f1` | One row per split |
| Card | `Precision at 0.35` (DAX above) |
| Card | `metrics` → `threshold`, filter `split = val` | Title: "Chosen threshold" |
| Scatter chart | X = `pred[PROB_REFUSED]`, Y = `pred[REFUSED]`, Size = 1, filter `SPLIT = test` | Optional; shows score separation |

## 4. Numbers to quote (from executed notebooks)

- Refusal rate overall **21.90%**; dev **17.28%**; OOT **37.26%**
- Model: LightGBM, threshold **0.35**
- test AUC **0.8281**, OOT AUC **0.7982**, OOT avg precision **0.7488**
- Top driver `NAME_CLIENT_TYPE` **25.1%** of gain; top 6 application features ≈ **86%**
- Refusal rate rose **17.35% → 37.39%** in the most recent 12 months

## 5. Optional: convert parquet to CSV

If Power BI cannot open `.parquet`, run this once in `01_data_preparation.ipynb` (or any notebook with `df` loaded) to emit a CSV version:

```python
# optional - only if Power BI cannot read parquet
pd.read_parquet(OUT + "stage1_modeling.parquet").to_csv(OUT + "modeling_table.csv", index=False)
print("wrote", OUT + "modeling_table.csv")
```

Then load `modeling_table.csv` as the `data` table instead. The CSV will be ~600 MB; Power BI handles it, but the parquet path is faster if available.
