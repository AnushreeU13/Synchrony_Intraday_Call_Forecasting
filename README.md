# Synchrony Financial Call Center Forecasting — Datathon 2026

## Problem Statement

Synchrony Financial operates four call centers — A, B, C, and D. The task was to forecast call center activity at a 30-minute interval level for the entire month of August 2025. For each half-hour slot, across each center, we needed to predict four things: how many calls would come in (Calls Offered), how many callers would hang up before being answered (Abandoned Calls), what fraction of callers would abandon (Abandon Rate), and how long the average call took (CCT — Customer Contact Time).

That works out to 31 days × 48 slots per day = **1,488 rows**, with 16 prediction columns (4 targets × 4 centers).

---

## Data Structure

All data came from a single Excel file with 9 sheets — one daily sheet and one interval sheet per center, plus a staffing sheet.

**Daily sheets (Centers A, B, C, D) — Jan 2024 to Dec 2025:**
Each row is one day. Columns are date, call volume, CCT, service level, abandon rate. This covers two full years and was the source of historical context features and — critically — the actual August 2025 daily totals that anchor our volume predictions.

**Interval sheets (Centers A, B, C, D) — April to June 2025 only:**
Each row is one 30-minute slot. Columns are month, day, interval time, service level, call volume, abandoned calls, abandon rate, CCT. This is the only interval-level data available and forms the basis for model training and validation.

**Daily Staffing — Jan to Dec 2025:**
One row per day, staffing counts for each center. Merged into the daily data as a feature.

---

## Code Files

### `01_preprocessing_final.ipynb`

Reads the raw Excel, cleans both the daily and interval data for all four centers, and saves processed files to S3.

**Daily preprocessing** starts by parsing the date column, which comes in a non-standard format like `"01/01/24 Mon"`. After parsing, it handles missing values in three different ways depending on severity. If only 1–2 metric columns are null in a row, it uses linear time interpolation — so the fill value is computed from the actual surrounding dates, not just a global average. If 3–4 columns are null, it finds the 3 closest same-weekday rows before the date and the 3 closest after, and fills from their average — again anchored to the specific period, not a blanket statistic. Rows where all metrics are null get dropped entirely. Staffing data is merged in at the end by date.

**Interval preprocessing** builds a proper datetime from three separate columns (month name, day number, and a time string like `"09:30:00"`). Missing slots are filled from the same slot 1 or 2 weeks prior — so a missing Tuesday 10:00am slot gets the value from the previous Tuesday's 10:00am slot, not from some overall average. If no prior week match exists, it falls back to the mean of all same-slot same-weekday rows.

**Output:** For each center, two clean CSVs are saved to S3:
- `daily_clean_{center}.csv` — one row per day, null-free
- `interval_merged_{center}.csv` — one row per 30-minute slot, with daily context columns merged in (daily call volume, daily CCT, daily service level, daily abandon rate, daily staffing). This merged file is the direct input to the modeling notebook.

---

### `02_eda_final.ipynb`

Exploratory analysis on the cleaned data. Key findings that informed modeling decisions: call volume drops roughly 50% on weekends; the intraday shape is remarkably stable across April, May, and June (slot-level correlations above 0.999); calls ramp sharply from 6am, plateau through midday, and drop after 5pm; abandon rate spikes at 6am when the center opens; and higher staffing clearly correlates with lower abandon rates. The CCT distribution showed that values above 600 seconds were rare outliers concentrated in overnight slots with near-zero call volume, which justified the 600-second cap applied in postprocessing.

---

### `03_model_doc_v68.ipynb`

The full modeling and prediction pipeline.

**Input features (19):**
slot_index, day_of_week, is_weekend, is_holiday, month, slot_mean_volume, slot_dow_mean_volume, daily_call_volume, daily_CCT, daily_service_level, daily_abandon_rate, daily_staffing, staffing_ratio, lag_1w_call_volume, lag_2w_call_volume, lag_1w_abandoned_calls, lag_1w_abandon_rate, lag_1w_CCT, lag_2w_CCT

**Model:**
Four separate LightGBM models are trained — one per target. Abandoned calls, abandon rate, and CCT use standard mean regression. These are the three targets whose predictions flow through directly to the submission file.

Call volume uses quantile regression at alpha = 0.55 — predicting the 55th percentile rather than the mean. The reason: the competition scoring formula penalises underprediction more than overprediction, so having a slight built-in upward bias is mathematically preferable to predicting the average. In practice though, the call volume model's prediction is discarded in postprocessing anyway (see below).

**Training and validation:**
Train on April 1 – June 15, validate on June 16 – June 30. After validation, retrain on the full April–June dataset before predicting August.

**Postprocessing — the most important part:**

*Call volume* is not predicted by the model. Instead: we compute each slot's historical share of the day's total call volume (its "proportion"), then multiply that proportion by the known August 2025 daily total. The daily totals for August were available in `daily_clean` — this became the key anchor. So the model's call volume output is thrown away entirely and replaced by:

> Calls in slot = slot_proportion × daily_actual_total × 1.05

The 1.05 multiplier was found through trial and error. We tested 1.00, 1.03, 1.04, and 1.05 via submission. Multiplier 1.00 matches the actuals most closely in raw terms, but 1.05 scores better — because being 5% above actual is better than being exactly right under the asymmetric penalty formula.

For the slot proportions themselves, we tried several aggregation methods and submitted each one:

| Method | Composite Score |
|---|---|
| Simple mean | 15.920694 |
| Median | 15.869518 |
| Trimmed mean 10% | 15.844159 |
| Trimmed mean 20% | 15.824385 |
| Trimmed mean 25% | 15.818039 |
| **Trimmed mean 40%** | **15.807925** |

The trimmed mean at 40% — dropping the most extreme 40% of days before averaging — consistently outperformed everything else. A single outlier day in training (an unusually high or low volume day) distorts a simple average. The trimmed mean is resistant to that. We also found that CCT proportions needed a different trim value than volume proportions — 25% for CCT vs 40% for volume — because call volume swings more day-to-day than handling time does.

*CCT* is also replaced in postprocessing — the model's CCT prediction is discarded and replaced with the trimmed mean CCT for that slot+weekday combination, capped at 600 seconds. The 600-second cap is data-driven: 97–99% of all training slots fall below this value, and the few that exceed it are overnight slots where a single long call in a near-empty period inflates the average unrealistically.

---

## Best Result

| Composite | Volume Error | CCT Error | Abandon Error | Workload Penalty |
|---|---|---|---|---|
| 15.807665 | 35.001537 | 13.96% | 1.48% | 0.134109 |

The 35% volume error is the floor we couldn't break through. June validation showed only 4–9% WMAPE — the gap is real and reflects genuine differences between August's intraday call patterns and the spring data we trained on.
