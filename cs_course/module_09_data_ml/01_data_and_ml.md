# Data and Machine Learning

Data engineering and machine learning have become core competencies for software engineers, not just specialists. Even if you never train a model yourself, you'll likely build systems that produce data for ML pipelines, integrate with ML models, or need to evaluate whether an ML solution is appropriate. This lesson covers the end-to-end lifecycle: from raw data ingestion to model serving in production.

---

## Why This Matters for Software Engineers

- **Data pipelines are software**: they have bugs, they need tests, they break at 3 AM
- **ML models are dependencies**: they have latency, failure modes, and version management — just like any other service
- **Garbage in, garbage out**: the most sophisticated model can't fix bad data. Understanding data quality is more valuable than knowing the latest algorithm
- **Decisions require context**: when a product manager asks "can we use ML for this?", you need enough understanding to give a useful answer

---

## Data Pipeline: Ingest, Clean, Transform, Serve

Every data system follows this fundamental flow, regardless of scale or technology:

```
[Sources]  →  [Ingest]  →  [Clean]  →  [Transform]  →  [Serve]
 APIs          Kafka        Dedupe      Aggregate       Data warehouse
 Databases     Firehose     Validate    Join             API
 Files         Webhooks     Normalize   Feature eng.     Dashboard
 Events        CDC          Handle      Aggregate        ML model
               (change      nulls
                data
                capture)
```

### Ingest

Getting data into your system from source systems:

```python
# Pull-based: periodically fetch from an API
def ingest_orders():
    response = requests.get("https://api.store.com/orders",
                            params={"since": last_checkpoint})
    for order in response.json():
        write_to_raw_storage(order)
    save_checkpoint(response.json()[-1]["created_at"])

# Push-based: receive events as they happen
# Kafka consumer
for message in consumer:
    event = json.loads(message.value)
    write_to_raw_storage(event)
```

**Key decisions**: pull vs push, frequency (real-time vs hourly vs daily), idempotency (processing the same event twice should be safe), backfill strategy (how to replay historical data).

### Clean

Raw data is messy. Cleaning transforms it into something trustworthy:

- **Deduplication**: the same event arriving twice (at-least-once delivery guarantees)
- **Null handling**: missing values — drop the row, fill with a default, or impute
- **Type coercion**: "42" as a string vs 42 as a number
- **Outlier detection**: a $-500 order or a user aged 250 — data entry errors or bugs
- **Schema validation**: does the data match the expected structure?

```python
def clean_order(raw: dict) -> dict | None:
    if raw.get("total") is None or raw["total"] < 0:
        log.warning(f"Invalid order total: {raw}")
        return None
    return {
        "order_id": str(raw["id"]),
        "total": float(raw["total"]),
        "created_at": parse_datetime(raw["timestamp"]),
        "customer_id": str(raw["customer_id"]),
    }
```

### Transform

Reshape cleaned data into a form optimized for consumption:

- **Aggregation**: daily revenue, hourly active users, conversion rates
- **Joins**: combine orders with customer profiles and product catalogs
- **Feature engineering**: derive new columns (days since last purchase, rolling average spend)
- **Denormalization**: flatten joins into wide tables for fast analytical queries

### Serve

Make the transformed data available to consumers:

- **Data warehouse** (BigQuery, Snowflake, Redshift): for analysts and dashboards
- **Feature store**: precomputed features for ML models
- **APIs**: serve data to applications
- **Reverse ETL**: push aggregated data back to operational systems (e.g., customer segments to a CRM)

---

## Batch vs Stream Processing

The two fundamental paradigms for processing data at scale:

### Batch Processing

Process data in large chunks on a schedule:

```
Raw data (files/tables)  →  [Batch Job]  →  Output (files/tables)
                             runs every hour/day
```

```sql
-- Example: daily aggregation in SQL (dbt-style)
SELECT
    date_trunc('day', created_at) AS order_date,
    COUNT(*) AS order_count,
    SUM(total) AS revenue,
    AVG(total) AS avg_order_value
FROM orders
WHERE created_at >= CURRENT_DATE - INTERVAL '1 day'
GROUP BY 1;
```

**Strengths**: simple to reason about, easy to rerun on failure, efficient for large volumes.
**Weaknesses**: data is always stale by up to one batch interval. Not suitable for real-time needs.
**Tools**: Spark, dbt, Airflow, simple cron + SQL scripts.

### Stream Processing

Process data record-by-record as it arrives:

```
Event stream (Kafka)  →  [Stream Processor]  →  Output (Kafka/DB/cache)
                          continuous
```

```python
# Conceptual stream processing (Flink/Kafka Streams style)
stream = kafka.consume("order_events")
stream \
    .filter(lambda e: e["status"] == "completed") \
    .map(lambda e: {"customer": e["customer_id"], "amount": e["total"]}) \
    .window(tumbling=timedelta(minutes=5)) \
    .aggregate(sum("amount")) \
    .sink("customer_spending_5min")
```

**Strengths**: low latency (seconds, not hours), reacts to events in real time.
**Weaknesses**: harder to debug, complex state management, exactly-once semantics are tricky, harder to reprocess historical data.
**Tools**: Kafka Streams, Apache Flink, Spark Structured Streaming.

### When to Use Which

| Factor | Batch | Stream |
|---|---|---|
| Freshness needed | Hours/days is fine | Seconds/minutes required |
| Complexity | Simpler | More complex |
| Reprocessing | Easy — rerun the job | Harder — replay the stream |
| Typical use | Reports, analytics, ML training | Alerts, fraud detection, live dashboards |

**Practical advice**: start with batch. Move to streaming only for use cases that genuinely need real-time data. Many "real-time" requirements are actually "within 15 minutes", which a frequent batch job handles just fine.

---

## Feature Engineering

Feature engineering is the process of turning raw data into inputs that make ML models more effective. It's often where the most impactful work happens — better features beat better algorithms.

### What Makes a Good Feature?

- **Predictive**: correlated with the target variable
- **Available at prediction time**: you can't use "did the user churn" as a feature to predict churn
- **Not leaky**: doesn't inadvertently include the answer (see Data Leakage below)
- **Computable**: can be calculated efficiently at both training and serving time

### Common Feature Engineering Techniques

```python
import pandas as pd

# Temporal features
df["hour_of_day"] = df["timestamp"].dt.hour
df["day_of_week"] = df["timestamp"].dt.dayofweek
df["is_weekend"] = df["day_of_week"].isin([5, 6])

# Aggregation features
df["user_order_count_30d"] = df.groupby("user_id")["order_id"] \
    .transform(lambda x: x.rolling("30D").count())

# Ratio features
df["return_rate"] = df["returns_count"] / df["orders_count"].clip(lower=1)

# Text features (simple)
df["description_length"] = df["description"].str.len()
df["description_word_count"] = df["description"].str.split().str.len()

# Encoding categorical variables
df = pd.get_dummies(df, columns=["category"])  # one-hot encoding
```

### Training-Serving Skew

A critical pitfall: features computed differently during training vs serving. If your training pipeline computes `avg_spend_30d` from a SQL query, but your serving pipeline computes it differently (different time window, different null handling), the model's predictions will be unreliable.

**Solution**: use a **feature store** (Feast, Tecton) that ensures the same feature computation logic is used for both training and serving.

---

## Model Types (High-Level)

You don't need to implement these from scratch, but you should understand what they're good at:

### Linear Models (Linear Regression, Logistic Regression)

- **How**: find a weighted sum of features that best predicts the target
- **Strengths**: fast, interpretable, works well with few features, good baseline
- **Weaknesses**: can't capture complex non-linear relationships
- **Use when**: predicting a number from a few features, binary classification with clear linear boundaries

### Decision Trees

- **How**: recursive if/else splits on features. "If age > 30 AND income > 50k, then approve"
- **Strengths**: interpretable, handles non-linear relationships, no feature scaling needed
- **Weaknesses**: overfit easily, unstable (small data changes → very different trees)

### Ensemble Methods (Random Forest, Gradient Boosting)

- **How**: combine many weak models (usually trees) into one strong model
- **Random Forest**: many trees trained on random subsets, averaged
- **Gradient Boosting** (XGBoost, LightGBM): trees trained sequentially, each correcting errors of the previous
- **Strengths**: top performance on tabular data, robust, handle missing values
- **Weaknesses**: less interpretable than single trees, can overfit without tuning
- **Use when**: tabular/structured data. This is the go-to for most real-world ML problems that don't involve images or text

### Deep Neural Networks

- **How**: many layers of interconnected neurons with non-linear activation functions
- **Strengths**: state-of-the-art for images (CNNs), text (Transformers), speech, and other unstructured data
- **Weaknesses**: require large datasets, expensive to train, hard to interpret, many hyperparameters
- **Use when**: unstructured data (images, text, audio), or when you have massive amounts of data

### Choosing a Model

```
Structured/tabular data?  → Start with gradient boosting (XGBoost/LightGBM)
Images?                   → Convolutional neural networks or pre-trained models
Text?                     → Transformer-based models (BERT, GPT-style)
Simple baseline needed?   → Linear/logistic regression
Need interpretability?    → Linear models or single decision trees
```

---

## Evaluation

### Train/Test Split

Never evaluate a model on the data it was trained on — it will look artificially good.

```python
from sklearn.model_selection import train_test_split

X_train, X_test, y_train, y_test = train_test_split(
    features, labels, test_size=0.2, random_state=42
)

model.fit(X_train, y_train)
predictions = model.predict(X_test)
```

**Cross-validation**: for more robust evaluation, split data into k folds, train on k-1, test on 1, rotate.

### Data Leakage

Data leakage is when information from the test set (or from the future) leaks into training, giving unrealistically good metrics:

- **Target leakage**: using a feature derived from the target. Example: using `account_closed_date` to predict churn — the date is only available *after* the churn happened
- **Train-test contamination**: normalizing features using statistics from the entire dataset (including test), or deduplicating after the split
- **Temporal leakage**: using future data to predict the past. Always split time-series data chronologically, not randomly

**If your model seems too good to be true, it probably has leakage.** A model that achieves 99.5% accuracy on a hard problem deserves skepticism, not celebration.

### Metrics

Different problems need different metrics:

| Problem Type | Metrics | When to Use |
|---|---|---|
| Binary classification | Accuracy, Precision, Recall, F1, AUC-ROC | Spam detection, fraud |
| Multi-class classification | Accuracy, Macro/Micro F1, Confusion matrix | Category prediction |
| Regression | MAE, RMSE, R², MAPE | Price prediction, forecasting |
| Ranking | NDCG, MAP, MRR | Search, recommendations |

**Critical**: accuracy is misleading for imbalanced datasets. If 99% of transactions are legitimate, a model that always predicts "legitimate" has 99% accuracy but catches zero fraud. Use precision/recall or AUC-ROC instead.

---

## Serving: Drift, Monitoring, and A/B Testing

Training a model is the beginning. Serving it reliably in production is the hard part.

### Model Serving Patterns

- **Online (real-time)**: model runs per request (e.g., fraud scoring at checkout). Low latency required
- **Batch**: model runs on a dataset periodically (e.g., nightly recommendations). Simpler, higher throughput
- **Near-real-time**: model results are precomputed and cached, refreshed frequently

### Model Drift

The real world changes. A model trained on pre-pandemic e-commerce data will perform poorly during a pandemic. There are two types:

- **Data drift**: the distribution of input features changes. Users start using the app differently, a new product category is added, seasonal patterns shift
- **Concept drift**: the relationship between features and the target changes. What predicted churn last year no longer predicts churn this year

**Detection**: monitor feature distributions and model output distributions over time. Alert when they diverge significantly from training data statistics.

```python
# Simple drift detection: compare feature distributions
from scipy.stats import ks_2samp

training_ages = training_data["age"]
production_ages = production_data["age"]

stat, p_value = ks_2samp(training_ages, production_ages)
if p_value < 0.05:
    alert("Age distribution has shifted — consider retraining")
```

### Model Monitoring

Monitor these in production:

- **Prediction latency**: is the model fast enough for real-time serving?
- **Prediction distribution**: are predictions shifting? If the model suddenly predicts 80% fraud (up from 2%), something is wrong
- **Feature distributions**: are inputs changing? Missing values increasing?
- **Business metrics**: are the downstream outcomes (conversion rate, revenue) moving in the expected direction?

### A/B Testing

The gold standard for evaluating whether a new model actually improves outcomes:

```
Users randomly split:
  Group A (50%): sees model v1 (control)
  Group B (50%): sees model v2 (treatment)

Measure: conversion rate, revenue, user engagement
Duration: long enough for statistical significance
```

**Pitfalls**:
- **Not running long enough**: small sample sizes produce unreliable results
- **Multiple comparisons**: testing many metrics increases false positive rate
- **Network effects**: if users interact (social networks), independent randomization doesn't work
- **Novelty effects**: users engage with new things just because they're new. Wait for the novelty to wear off

---

## Exercises

1. **Pipeline design**: you have a PostgreSQL database with user activity logs (page views, clicks, purchases). Design a data pipeline that produces a daily summary table with: user_id, total_purchases_30d, avg_session_duration_7d, last_login_days_ago. What tool would you use? How do you handle late-arriving data?

2. **Feature engineering**: given a dataset of e-commerce transactions (user_id, product_id, amount, timestamp, category), create 5 features that could predict whether a user will make a purchase in the next 7 days. For each feature, explain why it might be predictive and whether it has leakage risk.

3. **Leakage detection**: you're given a model that predicts customer churn with 99.2% accuracy. The features include: account_age, last_login_days_ago, support_tickets_count, cancellation_reason. Identify the leakage. How would you fix it?

4. **Batch vs stream decision**: for each scenario, decide whether batch or stream processing is more appropriate and justify:
   - Generating a weekly marketing email with personalized product recommendations
   - Detecting fraudulent credit card transactions
   - Computing daily revenue reports for the finance team
   - Updating a real-time leaderboard for an online game

5. **Drift monitoring plan**: you've deployed a model that predicts delivery time for an e-commerce platform. What metrics would you monitor? What would trigger a retrain? How would you A/B test a new version?

---

## Recommended Resources

- **"Designing Data-Intensive Applications" by Martin Kleppmann** — chapters on batch and stream processing are the best explanation of data pipelines available.
- **"Designing Machine Learning Systems" by Chip Huyen** — covers the full ML lifecycle from a systems perspective: data, features, training, serving, monitoring.
- **"Machine Learning Engineering" by Andriy Burkov** — practical guide focused on the engineering side of ML, not the math.
- **Google's Rules of ML ([developers.google.com/machine-learning/guides/rules-of-ml](https://developers.google.com/machine-learning/guides/rules-of-ml))** — battle-tested guidelines from Google's ML teams. Start here.
- **Made With ML ([madewithml.com](https://madewithml.com))** — end-to-end MLOps course covering data, models, and production deployment.
- **"The Hundred-Page Machine Learning Book" by Andriy Burkov** — concise, high-density introduction to ML concepts without the textbook bloat.
