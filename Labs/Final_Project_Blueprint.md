Business Problem: 

We will build a predictive earthquake-risk analytics system that combines the real-time USGS Earthquake Catalog API with decades of historical seismic activity from the Kaggle earthquake database. This project will clean, merge, and model both data sources to identify patterns in magnitude, depth, and regional frequency, enabling better detection of high-risk zones. By delivering actionable insights, the system helps emergency-management teams, infrastructure planners, and insurers better anticipate seismic threats, allocate resources more efficiently, and make data-driven decisions that reduce potential loss of life and economic damage. 

 

 

Architecture Design: 

🛠️ Pipeline Flow Breakdown 

1. 📂 Historical Data (Batch) Flow 

The 23,000-record CSV file is uploaded to a Cloud Storage bucket. 

A Dataflow job (configured for Batch mode) is launched. 

The Dataflow job reads the CSV, applies required schema transformations, and writes the clean data to the Historical Earthquakes table in BigQuery. 

2. ⚡ Real-Time Data (Streaming) Flow 

A dedicated Cloud Function is configured to receive and process events from the external streaming API. 

Upon receiving a live earthquake event, the Cloud Function formats the data and publishes it as a message to a Pub/Sub topic. 

A separate Dataflow job (configured for Streaming mode) subscribes to the Pub/Sub topic. 

Dataflow continuously processes the incoming messages, applies any necessary real-time calculations (like alert level logic), and streams the results into the Live Earthquakes table in BigQuery. 

3. 📊 Dashboarding 

Looker Studio connects directly to the BigQuery dataset. 

You build visualizations using SQL queries against the BigQuery tables to display real-time magnitude metrics, alert levels, and historical trends. 

 

 

Machine Learning Component: 

Machine Learning Plan (BigQuery ML) 

Problem and target. 
 We’ll treat this as a binary classification problem at the region–time-window level (for example, a 0.5°×0.5° grid cell over a 30-day window). The label high_risk = 1 if that region has at least one M ≥ 5.5 event in the next 30 days, and 0 otherwise. The goal is not exactly predicting individual quakes, but a relative near-term risk score to rank zones for planners and insurers. 

Data sources and features. 

Batch (historic) features – Kaggle CSV 
 From the USGS earthquake database on Kaggle (date, time, latitude, longitude, depth, magnitude, etc.Kaggle), we’ll compute long-term, slowly changing features per region: 

Long-run event rate (events / year), total event count. 

Historical max and 95th-percentile magnitude. 

Average and standard deviation of depth; share of shallow events (< 70 km). 

Inter-event time statistics (median days between M ≥ 4 events). 

Streaming features – USGS Earthquake Catalog API 
 Using the USGS FDSN Earthquake Catalog API (https://earthquake.usgs.gov/fdsnws/event/1/query USGS+1) we’ll stream recent events into BigQuery and aggregate per region: 

Counts of events in the last 1, 7, and 30 days. 

Max and average magnitude/depth in the last 30 days. 

Time since last M ≥ 4.5 event. 

Short-term vs long-term activity ratio (recent 30-day count / typical 30-day count). 

These will be joined into a single training table of region + reference date with past features and a future label. 

Model choice and training. 
We’ll first train a logistic regression model in BigQuery ML for transparency, with the option to later compare a BOOSTED_TREE_CLASSIFIER: 

CREATE OR REPLACE MODEL earthquakes_ml.earthquake_risk_model 
OPTIONS( 
  model_type = 'logistic_reg', 
  input_label_cols = ['high_risk'] 
) AS 
SELECT 
  * 
FROM 
  earthquakes_ml.training_features; 

Training data will be built from rolling windows across multiple years of history. Because high-magnitude events are rare, we’ll address class imbalance either with BQML’s class weighting (if supported for the chosen model) or by oversampling positive cases in the training query. 

Evaluation and threshold. 
 Using ML.EVALUATE on a hold-out time slice (e.g., the last 2–3 historical years), we’ll track: 

AUC, precision, recall, and log loss. 

Precision–recall trade-offs across thresholds (0.1, 0.2, 0.3, etc.). 

We’ll pick a classification threshold that favors high recall for true high-risk regions while keeping false alarms manageable, and explicitly document that threshold for downstream users. 

Explainability and integration. 

Use ML.EXPLAIN_PREDICT on sample regions to show which features (e.g., recent 30-day event count vs long-term max magnitude) drive risk scores, supporting communication to non-technical stakeholders. 

On a schedule (e.g., every 30–60 minutes), a BigQuery job will: 

Refresh recent streaming features from the USGS API data. 

Join with static Kaggle-based region features. 

Call ML.PREDICT on earthquake_risk_model and write updated risk scores per region to earthquakes_ml.region_risk_scores. 

That prediction table will be the source for our Looker Studio dashboard, enabling near-real-time maps and time-series of earthquake risk by region. 

 

 

 

Final Looker Studio Dashboard: 

Dashboard KPIs (Looker Studio) 

Our Looker Studio dashboard will focus on 3–5 core KPIs: 

Current High-Risk Regions Count 
 Number of regions with predicted high_risk = 1 (risk score above the chosen threshold) in the latest run. 

Top-N Regions by Risk Score 
 Table/map of the N regions with the highest predicted probability of an M ≥ 5.5 event in the next 30 days. 

Risk vs Historical Baseline 
 For a selected region, comparison of current risk score and recent 30-day activity against its long-term historical average. 

Recent Seismic Activity Index 
 Rolling index showing counts and average magnitude of events over the last 1, 7, and 30 days for each region (or globally). 

Model Performance Snapshot (Optional) 
 Over a recent backtest window: precision, recall, and number of correctly flagged high-risk regions to give stakeholders confidence in the model. 

 

 

Challenge Architecture: 

⚠️ Single Biggest Risk: Backlog and Processing Latency 

The core of the issue lies in the Pub/Sub -> Dataflow -> BigQuery path: 

Pub/Sub Decoupling is a Double-Edged Sword: While Pub/Sub provides durability and can buffer messages, if your incoming message rate (Publishing) consistently exceeds the rate at which Dataflow can process and write those messages (Subscribing), the message backlog in Pub/Sub will grow indefinitely. 

Consequence: A growing backlog means the "real-time" analysis displayed in your Looker Studio dashboard will lag further and further behind the actual live earthquake events. If the lag is too great, the alerts and metrics become useless for immediate operational response. 

Dataflow Autoscaling Limits: Although Dataflow autoscales, there are limits to how quickly and how much it can scale. Intensive transformations, slow sink write speeds (e.g., if BigQuery is under heavy concurrent load), or inefficient Beam code can all throttle its throughput. 

 

🛡️ Mitigation Strategy: Throughput Testing and Resource Pre-sizing 

The best way to mitigate this risk before building is to establish a rigorous, data-driven understanding of your pipeline's capacity. 

1. Establish a Peak Load Baseline 

Identify Maximum Expected Rate: Determine the absolute maximum message rate (messages/second or MB/second) you expect from your streaming API during a major event. (e.g., 500 messages/sec). 

Define Latency SLO: Determine the maximum acceptable delay from the moment an event hits Pub/Sub to when it appears in BigQuery (e.g., P95 latency must be under 10 seconds). 

2. Run a Pre-Deployment Load Test 

Simulate Load: Use a tool (or a dedicated script running on Compute Engine) to pump simulated data into your Pub/Sub topic at 2 times your maximum expected rate. 

Test Dataflow Performance: Deploy a prototype Dataflow streaming job with your actual sentiment analysis logic. Do not rely on simplified logic for this test. 

Monitor Backlog and Latency: Monitor two key metrics: 

Pub/Sub Undelivered Messages: Is this number growing? 

Dataflow System Latency: Does the end-to-end latency exceed your defined SLO? 

3. Resource Pre-Sizing and Optimization 

Based on the test results, you can make necessary adjustments: 

Dataflow Machine Type: If your logic is CPU-intensive (e.g., complex sentiment models), pre-size your Dataflow workers to use larger, higher-CPU machine types to maximize throughput per worker. 

BigQuery Optimizations: Ensure your BigQuery sink writes are batched efficiently within Dataflow to minimize the overhead of too many small, individual insertions. Consider using clustering or partitioning on time fields to optimize insertion performance. 

 

 

 

 

Data Sources: 

API: https://earthquake.usgs.gov/fdsnws/event/1/ (Earthquake Catalog API) 

Kaggle: https://www.kaggle.com/datasets/usgs/earthquake-database (historical CSV file) 

 
