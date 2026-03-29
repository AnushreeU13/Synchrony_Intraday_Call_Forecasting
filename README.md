# Synchrony Financial Call Center Forecasting - Datathon 2026

I was given historical call center data across 4 centers (A, B, C, D) and asked to predict what August 2025 would look like - specifically, how many calls would come in, how many would be abandoned, what the abandon rate would be, and how long calls would take, all broken down into 30-minute slots across the entire month.

That's 31 days × 48 slots = **1,488 rows of predictions**, for each of the 4 centers.

---

## What's in here

| File | What it does |
|---|---|
| `01_preprocessing_v68.ipynb` | Reads the raw Excel, cleans it up, handles missing data, merges in staffing numbers, and saves everything to S3 ready for modeling |
| `02_eda_v68.ipynb` | Exploratory analysis - I looked at volume trends over time, intraday call patterns, weekday vs weekend differences, and what drives abandon rates |
| `03_model_doc_v68.ipynb` | The full modeling pipeline - from feature engineering to predictions to the final submission file |

Note: I had time to edit the code files and I perfected 03_model_doc_v68.ipynb - **IT'S NOT AI** I just spent time decorating it unlike the other 2 files.
---

## The Data

Everything came from one Excel file: `s3://synchrony-callcenter/raw/Datathon_Data.xlsx`

| Sheet | Date range | How I used it |
|---|---|---|
| A/B/C/D - Daily | Jan 2024 – Dec 2025 | Daily context for the model + the actual August 2025 daily totals that anchor our predictions |
| A/B/C/D - Interval | Apr – Jun 2025 | The 30-min slot level data I trained and validated on |
| Daily Staffing | Jan – Dec 2025 | Staffing numbers merged in as a feature |

A couple of terms I use throughout:
- **Slot** - one 30-minute window. Slot 0 is midnight, slot 47 is 11:30pm, 48 slots per day.
- **DoW** - day of week. 0 = Monday, 6 = Sunday.

---

## How I approached it

The honest answer is that I tried a lot of things before landing on what worked.

Early on I let LightGBM predict call volume directly - it wasn't great. The big breakthrough was realising I already had the actual August 2025 daily totals in `daily_clean`, so instead of predicting daily volume from scratch, I just needed to figure out how to split each day's total across the 48 slots. That's the slot proportion approach - for each slot+weekday combination, I compute what fraction of the day's calls typically land in that slot, then multiply by the known daily total.

For the proportions themselves, I found that a 40% trimmed mean (dropping the most extreme 40% of days before averaging) was more robust than a plain average or median. A single unusually busy or quiet day in the training data can distort a simple average - the trimmed mean is less sensitive to that.

CCT worked the same way - replace the model's prediction with a trimmed mean of historical CCT per slot+weekday, capped at 600 seconds (10 minutes, which the data showed was a reasonable upper bound for anything other than extreme overnight outliers).

That left abandoned calls and abandon rate as the only two targets where the LightGBM model's output actually flows through to the submission. I trained separate models per target - one quantile regression model for call volume (even though its output gets replaced in postprocessing - this was a legacy of the architecture that produced our best score) and standard mean regression for everything else.

The 1.05 multiplier on call volume came from observing that the scoring formula penalises underprediction more than overprediction. Being 5% above actuals consistently scored better than being spot on.

---

## Running it

Run the notebooks in order:

```
1. 01_preprocessing_v68.ipynb
2. 02_eda_v68.ipynb        ← optional, EDA only
3. 03_model_doc_v68.ipynb
```

You'll need:
```
pip install holidays openpyxl lightgbm scikit-learn scipy pandas numpy boto3
```

Runs on AWS SageMaker with access to `s3://synchrony-callcenter/`.

---

## Best result

| Composite | Volume Error | CCT Error | Abandon Error | Workload Penalty |
|---|---|---|---|---|
| 15.807665 | 35.001537 | 13.96% | 1.48% | 0.134109 |

Volume error dominates the composite score (it carries ~45% of the weight). The 35% error is largely irreducible given i only had April–June interval data to learn from - the intraday shape is stable across those months, but August's slot-level distribution differs from spring in ways the training data can't fully capture.
