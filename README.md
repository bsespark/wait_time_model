# Wait Time Prediction Model (before property separation) 

## 📊 1. Data Sources & Cleaning
- **Snowflake connection** to pull core wait time data.
- Removed irrelevant events: `Test`, `Cancelled`, `Upsell`.
- Defined **relative start time cutoff** and created `5_minute_bucket` for temporal grouping.
- Added **queue trend features**: 5 minute --> 3 minute 
  - `queue_mean_3`: 3-minute rolling mean of line size 
  - `queue_slope_3`: Rolling trend indicating if the line is growing/shrinking.
- Added **cyclical time encodings** (`hour_sin/cos`, `day_sin/cos`, `minute_sin/cos`).
- Integrated **attendance data** (no missing).
- Integrated **tier data**:
  - Missing Liberty/Nets tiers estimated from opponent averages.
  - Missing Barclays tiers estimated from attendance quantiles. 
- Added **weather data**: historical & forecast (`temperature_2m_max/min`, `precipitation_sum`)
  - Forecast will be added in a separate file for production.
- Attempted to add **external event context**: Ticketmaster, Eventbrite, Seatgeek nearby events + NYC traffic data.
  - TM possible, need more time for other features 
- Verified **no missing columns** in final feature set.

--

## 🧮 2. Features for Modeling
**Categorical**  
`event_day`, `team`, `event_type`, `minor_category`, `property`, `time category`

**Numerical**  
`5_minute_bucket`, `queue_mean_5`, `queue_slope_5`, `hour`, `hour_sin`, `hour_cos`,  
`day_sin`, `day_cos`, `minute_sin`, `minute_cos`, `count`, `tier`,  
`temperature_2m_max`, `temperature_2m_min`, `precipitation_sum`
#`nyc_event_count`, `traffic_index`

---

## 🤖 3. Model Setup
- **Quantile Regression** for worst-case wait times at the 95th percentile (Q95).
- Models tested:
  - `GradientBoostingRegressor` (baseline Q95).
- Loss function: **Pinball loss** (penalizes under-predictions more).
- Group-wise split by `event_name` to prevent data leakage.
- Preprocessing:
  - Numerical: `StandardScaler`
  - Categorical: `OneHotEncoder`

---

## 📏 4. Evaluation Metrics
v3 Model Evaluation Metrics (Q95)
-----------------------------------
**Mean Absolute Error (MAE)**:       3.01 min
**Coverage**:                       **86.6%** vs target of 95%, meaning ~8.4% of minutes exceed the predicted Q95
Worst misses occur during **true peak demand** (major NYC events, unusual weather, sudden pre-event surges).
**Pinball Loss**:                     **0.3271**, indicating both underestimation in the tail and unnecessary inflation in non-peak minutes.
**Over-Prediction Rate**:           **86.6%**, average buffer = **+3.25 min**.
Model provides a consistent safety margin in most minutes but **wastes buffer on low-risk periods**.
**Average Safety Buffer**:           3.25 min 

---

## 🔍 5. Error Patterns & Insights (v3)

**Feature-Driven Overestimation**  
  - SHAP shows features like `minute_sin/cos`, `queue_mean_3`, and `queue_slope_3` push predictions up during typical queue growth patterns.
  - This inflates predictions in moderate build-ups but doesn’t fully capture **extreme, rare spikes**.

---

## 🚀 7. Next Steps

1. **Tail Calibration**  ??
   - Apply isotonic regression or Platt scaling to better align predicted Q95 with actual 95th percentile coverage.

2. **Extreme-Event Feature Engineering**  
   - Integrate features that better capture rare spikes:
     - Ticket sell-through % / day-of-game sales ???
     - Same-day transit disruptions
     - Door opening delays or operational incidents / ingress delays 
     - High-impact concurrent NYC events

3. **Dynamic Safety Buffering**  
   - Use conditional features (e.g., queue acceleration thresholds) to **allocate extra buffer only during true risk periods**.

4. **Minute Bucket Error Profiling**  
   - Conduct bucket-level diagnostics to identify specific `5_minute_bucket` ranges with the highest under-prediction.

5. **Model Variants by Context**  
   - Train separate Q95 models for:
     - High-tier vs low-tier events
     - Weather-impacted vs clear days
     - Weekday vs weekend

6. **Future Integration with LightGBM**  
   - Once new extreme-event features are added, re-run with LightGBM to test multi-quantile predictions for richer operational insights.

---

## 📜 9. Code Execution Order
1. Connect to Snowflake & pull wait time data.  
2. Filter irrelevant events.  
3. Create `5_minute_bucket` & relative time cutoffs.  
4. Add queue trend features (`queue_mean_3`, `queue_slope_3`).  
5. Add cyclical encodings for time variables.  
6. Merge attendance & tier data (fill missing values).  
7. Add weather & external event data.  
8. Preprocessing pipeline (scale numeric, encode categorical).  
9. Train Gradient Boosting.  
10. Evaluate with coverage, pinball, MAE.  
11. Residual/error diagnostics by category & time bucket.
