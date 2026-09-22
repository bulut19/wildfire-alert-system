# Wildfire Alert System

A cloud data pipeline that combines 24 years of historical wildfire records with live weather data to predict fire severity and potential acreage burned under current conditions, built on a Lambda architecture in Google Cloud.

## Overview

The system answers a specific operational question: given current wind, temperature, and humidity, how severe would a fire be if one started right now. Three layers work together:

- **Batch layer.** 1.88 million historical U.S. wildfire records (1992-2015, Kaggle) are loaded into Cloud Storage and then BigQuery, forming the training data for both models.
- **Speed layer.** Live weather is pulled from the Open-Meteo API on a schedule, published to Pub/Sub by a Cloud Function, and streamed into BigQuery.
- **Analytics layer.** Two BigQuery ML models run on the combined data: a regression model estimating acres burned, and a classification model assigning a severity category. Predictions are generated across a grid of coordinates spanning California and visualized on a Looker Studio dashboard.

![Wildfire risk dashboard showing KPI tiles, a risk-level breakdown, a high-risk areas table, and a California heatmap of predicted acres burned](dashboard_screenshot.png)
*The live Looker Studio dashboard is no longer reachable (its BigQuery credentials expired), so a screenshot is included in its place.*

## Key Findings

- **The linear regression baseline failed, and that failure is documented rather than hidden.** Predicting acres burned (`FIRE_SIZE`) directly, it produced an R2 of just 0.08, MAE of 4,046 acres, and RMSE of 15,766 acres.
- **A Boosted Tree Regressor with feature engineering cut error by more than half.** Capping the target at its 99th percentile and adding engineered features (a temperature-humidity interaction term, a fire danger index, and two geographic features) brought MAE down to 1,603.94 acres and RMSE to 4,856.73, with R2 improving to 0.34, a fourfold gain over the linear baseline.
- **The classification model reaches strong overall accuracy but weaker per-class precision.** A logistic regression classifying fire severity (Low/Medium/High) achieved 89.7% accuracy and a 0.865 ROC AUC, but precision (0.578), recall (0.405), and F1 (0.426) were considerably lower, meaning it's better at overall discrimination than at reliably catching every high-severity event.
- **The streaming pipeline closes the loop end to end.** Live weather flows through a Cloud Function, Pub/Sub, and a Dataflow job into BigQuery, where it's combined with a generated grid of California coordinates and scored by both models to produce a continuously updated risk surface.

## Limitations

1. The regression model's R2 of 0.34 means most of the variance in fire size remains unexplained; predicted acreage should be read as a directional estimate, not a precise forecast.
2. The classification model's precision and recall on the Medium and High severity classes are meaningfully lower than its headline accuracy, which is driven largely by the more common Low-severity class.
3. The live weather feed is scoped to a single reference point (Los Angeles) as a proof of concept for the ingestion pipeline; polling each region in the prediction grid, so the heatmap reflects real spatial weather variation, is the most direct next step toward a production version.
4. Historical training data spans 1992-2015 and isn't reweighted for recent climate trends, and its regional coverage (concentrated in California and other historically fire-prone states) can bias predictions for underrepresented areas.
5. No automated monitoring or retraining is in place; both models would need a periodic retraining cadence in production to avoid drift.

## Tech Stack

Google Cloud (Cloud Storage, BigQuery, BigQuery ML, Pub/Sub, Cloud Functions, Cloud Scheduler, Dataflow) · Looker Studio · Open-Meteo API · Python (pandas, NumPy, matplotlib, seaborn) · Google Colab

## Repository Contents

- `wildfire_alert_system.ipynb`: full pipeline (batch ingestion, both ML models with the linear-to-boosted-tree progression, real-time streaming infrastructure, and grid-based prediction generation for the dashboard)
- `README.md`: this file
- `dashboard_screenshot.png`: screenshot of the Looker Studio dashboard

## Team

Group project with Gabriel Wang and Xinyi Zhang.
