# BigQuery ML Demo
Getting started with BigQuery ML: a pragmatic approach on how to get started with Machine Learning on GCP via BigQuery

## Getting started 
[Log in to BigQuery](https://console.cloud.google.com/bigquery)

... that's it! 

## Time Forecasting example
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

### Step 4: Run predictions and explain them
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






## Further Readings
- [End-to-end user journey](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-syntax-e2e-journey)
- [The CREATE model statement](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-syntax-create)
- [Modeling in BigQuery ML](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-syntax-e2e-journey)
