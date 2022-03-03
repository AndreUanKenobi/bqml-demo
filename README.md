# My very first BigQuery ML Demo
Getting started with BigQuery ML: a pragmatic approach on how to get started with Machine Learning on GCP via BigQuery. This Demo will cover three use cases:
- [Prediction via Time Forecasting](https://github.com/AndreUanKenobi/bqmldemo/edit/main/README.md#time-forecasting-example)
- [Classification via Logistic Regression](https://github.com/AndreUanKenobi/bqmldemo/edit/main/README.md#logistic-regression-model)
- [Unsupervised Anomaly Detection](https://github.com/AndreUanKenobi/bqmldemo/edit/main/README.md#unsupervised-anomaly-detection)

Ready to get started? Let's [log in to BigQuery](https://console.cloud.google.com/bigquery)

## Time Forecasting
[Reference](https://cloud.google.com/bigquery-ml/docs/arima-single-time-series-forecasting-tutorial)

### Step 1: Visualize the data
Run the following statement in BigQuery:
```
#standardSQL
SELECT
  PARSE_TIMESTAMP("%Y%m%d", date) AS parsed_date,
  SUM(totals.visits) AS total_visits
FROM
  `bigquery-public-data.google_analytics_sample.ga_sessions_*`
GROUP BY date
```
And [analyze the results in Data Studio](https://cloud.google.com/bigquery-ml/docs/arima-single-time-series-forecasting-tutorial#optional_step_two_visualize_the_time_series_you_want_to_forecast)

### Step 2: Create an ARIMA model
Run the following statement in BigQuery:
```
#standardSQL
CREATE OR REPLACE MODEL bqml_tf.ga_arima_model
OPTIONS
  (model_type = 'ARIMA_PLUS',
   time_series_timestamp_col = 'parsed_date',
   time_series_data_col = 'total_visits',
   auto_arima = TRUE,
   data_frequency = 'AUTO_FREQUENCY',
   decompose_time_series = TRUE
  ) AS
SELECT
  PARSE_TIMESTAMP("%Y%m%d", date) AS parsed_date,
  SUM(totals.visits) AS total_visits
FROM
  `bigquery-public-data.google_analytics_sample.ga_sessions_*`
GROUP BY date
```
The auto.arima() function automates the estimation of a constant (if necessary) in order to improve the resulting [Akaike's Information Criterion (AIC)](https://www.scribbr.com/statistics/akaike-information-criterion/) value

### Step 3: Evaluate the model
Run the following statement in BigQuery:
```
#standardSQL
SELECT
 *
FROM
 ML.ARIMA_EVALUATE(MODEL bqml_tf.ga_arima_model)
```
Several quality parameters are returned: 
- p: the number of lag observations in the model; also known as the lag order
- d: the number of times that the raw observations are differenced; also known as the degree of differencing
- q: the size of the moving average window; also known as the order of the moving average.
- drift: think of it as an intercept
- Akaike's Information Criterion (AIC): lower AIC values indicate a better-fit model

### Step 4: Run predictions and explain the model
Run the following statement in BigQuery to create predictions:
```
#standardSQL
SELECT
 *
FROM
 ML.FORECAST(MODEL bqml_tf.ga_arima_model,
             STRUCT(30 AS horizon, 0.8 AS confidence_level))
```

Run the following statement in BigQuery to explain the model:
```
#standardSQL
SELECT
 *
FROM
 ML.EXPLAIN_FORECAST(MODEL bqml_tf.ga_arima_model,
                     STRUCT(30 AS horizon, 0.8 AS confidence_level))
```

## Classification via Logistic Regression
[Reference](https://cloud.google.com/bigquery-ml/docs/logistic-regression-prediction)

### Step 1: Visualize the data
Run the following statement in BigQuery:
```
SELECT 
  *
FROM
  `bigquery-public-data.ml_datasets.census_adult_income`
LIMIT
  100;
```
### Step 2: Prepare the train, validation and prediction datasets
Run the following statement in BigQuery:
```
CREATE OR REPLACE VIEW
  `bqml_logreg.input_view` AS
SELECT
  age,
  workclass,
  native_country,
  marital_status,
  education_num,
  occupation,
  race,
  hours_per_week,
  income_bracket,
  CASE
    WHEN MOD(functional_weight, 10) < 8 THEN 'training'
    WHEN MOD(functional_weight, 10) = 8 THEN 'evaluation'
    WHEN MOD(functional_weight, 10) = 9 THEN 'prediction'
  END AS dataframe
FROM
  `bigquery-public-data.ml_datasets.census_adult_income`

```

### Step 3: Create the model
Run the following statement in BigQuery:
```
CREATE OR REPLACE MODEL
  `bqml_logreg.census_model`
OPTIONS
  ( model_type='LOGISTIC_REG',
    auto_class_weights=TRUE,
    data_split_method='NO_SPLIT',
    input_label_cols=['income_bracket'],
    max_iterations=15) AS
SELECT
  * EXCEPT(dataframe)
FROM
  `bqml_logreg.input_view`
WHERE
  dataframe = 'training'

```

### Step 4: Evaluate the model
Run the following statement in BigQuery:
```
SELECT
  *
FROM
  ML.EVALUATE (MODEL `bqml_logreg.census_model`,
    (
    SELECT
      *
    FROM
      `bqml_logreg.input_view`
    WHERE
      dataframe = 'evaluation'
    )
  )

```

Where:
- Precision: True positives / (True Positivies + False Positives) 
- Recall: True positives / (True positives + False negatives)

_ _Precision can be seen as a measure of quality, and recall as a measure of quantity. Higher precision means that an algorithm returns more relevant results than irrelevant ones, and high recall means that an algorithm returns most of the relevant results (whether or not irrelevant ones are also returned)._ _

- F1: ( Precision * Recall) / (Precision + Recall); it's a [harmonic mean](https://en.wikipedia.org/wiki/Harmonic_mean)
- Log loss: log (product of all likelyhoods of predictions), the lower the better
- [ROC_AUC](https://glassboxmedicine.files.wordpress.com/2019/02/roc-curve-v2.png?w=576): Area under ROC (Receiver operating characteristic) curve> the higher, the better

### Step 5: Run predictions
Run the following statement in BigQuery:
```
SELECT
  *
FROM
  ML.PREDICT (MODEL `bqml_logreg.census_model`,
    (
    SELECT
      *
    FROM
      `bqml_logreg.input_view`
    WHERE
      dataframe = 'prediction'
     )
  )
```


## Unsupervised Anomaly Detection
[Reference](https://cloud.google.com/blog/products/data-analytics/bigquery-ml-unsupervised-anomaly-detection)

We will be using two different appraches: 
- Machine Learning oriented, via K-Means
- Deep Neural Networks oriented, via Autoencoders

### Step 1: Visualize the data
Run the following statement in BigQuery:
```
SELECT 
  * 
FROM 
  `bigquery-public-data.ml_datasets.ulb_fraud_detection`
  limit 100;
```

### Step 2: Create the model
Using K-means:
```
CREATE MODEL `bqml_anomaly.my_kmeans_model`
OPTIONS(
  MODEL_TYPE = 'kmeans',
  NUM_CLUSTERS = 8,
  KMEANS_INIT_METHOD = 'kmeans++'
) AS
SELECT 
  * EXCEPT(Time, Class) 
FROM 
  `bigquery-public-data.ml_datasets.ulb_fraud_detection`;

```
Using Autoencoder: 
```
CREATE MODEL `bqml_anomaly.my_autoencoder_model`
OPTIONS(
  model_type='autoencoder',
  activation_fn='relu',
  batch_size=8,
  dropout=0.2,  
  hidden_units=[32, 16, 4, 16, 32],
  learn_rate=0.001,
  l1_reg_activation=0.0001,
  max_iterations=10,
  optimizer='adam'
) AS 
SELECT 
  * EXCEPT(Time, Class) 
FROM 
  `bigquery-public-data.ml_datasets.ulb_fraud_detection`;

```

### Step 3: Detecting anomalies
Using K-means:
```
SELECT
  *
FROM
  ML.DETECT_ANOMALIES(MODEL `bqml_anomaly.my_kmeans_model`,
                      STRUCT(0.02 AS contamination),
                      TABLE `bigquery-public-data.ml_datasets.ulb_fraud_detection`);
```

Using Autoencoder: 
```
SELECT
  *
FROM
  ML.DETECT_ANOMALIES(MODEL `bqml_anomaly.my_autoencoder_model`,
                      STRUCT(0.02 AS contamination),
                      (SELECT * FROM `bigquery-public-data.ml_datasets.ulb_fraud_detection`));
```

Note: in both cases, the **contamination** parameter drives the anomaly cut-off

## Further Readings
- [End-to-end user journey](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-syntax-e2e-journey)
- [The CREATE model statement](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-syntax-create)
- [Modeling in BigQuery ML](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-syntax-e2e-journey)
