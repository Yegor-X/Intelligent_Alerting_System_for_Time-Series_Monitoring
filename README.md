# Intelligent Alerting System for Time-Series Monitoring

## Overview
This repository contains a robust, decoupled machine learning pipeline designed to predict and alert on system anomalies (such as server outages) in time-series data. 

The primary goal of this project is to solve the Alert Fatigue problem. By separating the raw anomaly prediction from the alerting business logic, this system maintains a high recall (catching the vast majority of incidents) while drastically reducing false positives (spam alerts) that typically overwhelm DevOps engineers.

## Architecture
The system is built on a Decoupled Architecture, breaking the problem into two distinct layers:

1. The Predictor: A machine learning classification model that analyzes time-series windows and outputs a raw probability of an impending incident.
2. The Decision Engine: A stateful, learning-augmented risk manager that processes the probabilities stream, accumulates risk over time, and enforces silence periods to prevent alert spam.



## How the Code Works:

### 1. Synthetic Data Generation
The function `generate_synthetic_data` creates a realistic time-series dataset simulating server metrics (like CPU utilization). 
- It includes a daily sinusoidal seasonality and random noise.
- Anomalies are injected with realistic precursors. Instead of just jumping from normal to failing, the metric slowly degrades (gradual increase over 25 minutes) before the actual hard spike occurs. This allows the model to learn early-warning signs.

### 2. Smart Feature Engineering
Raw metric values are heavily dependent on the time of day and baseline drift. Feeding raw arrays to an ML model often leads to overfitting. The `extract_smart_features` function solves this by providing context:
- It uses a sliding window approach (e.g., 60 minutes).
- It calculates standard statistics: Max, Mean, Standard Deviation.
- Dynamic Features: It calculates the Z-Score and the relative change by comparing the last 10 minutes against the full 60-minute window. This teaches the model to look for sudden deviations from the current local norm, rather than hardcoded threshold values.

### 3. The Predictor Model
The system uses a `RandomForestClassifier` as the Predictor. 
- It is trained to predict whether an incident will occur within the next H minutes (Prediction Horizon).
- Class Imbalance Handling: The model uses `class_weight={0: 1, 1: 10}`. This intentionally biases the model to prioritize catching anomalies, heavily penalizing false negatives. The resulting over-sensitivity is then filtered out by the Decision Engine.

### 4. The Decision Engine (Smart LAA)
This is the core innovation of the project. A naive baseline would trigger an alert the moment the Random Forest outputs a probability above a certain threshold (e.g., 0.3). This causes massive spam.

Instead, the `SmartAlerter` class implements a risk accumulation logic (inspired by Ski-Rental and Learning-Augmented Algorithms):
- Minimum Threshold: If the model's probability is below `start_prob` (e.g., 0.5), it is ignored.
- Risk Accumulation: If the probability is high, the engine adds it to an internal `acc_risk` counter. An alert is only fired if this accumulated risk breaches the `risk_limit`. This means the ML model must be confident for several consecutive minutes before an engineer is paged.
- Cooling Down: If the probability drops back to normal, the accumulated risk slowly decays.
- Cooldown Period: Once an alert fires, the system enters a strict silence period (`cooldown=45` minutes). This guarantees only one notification per incident block.

### 5. Event-Based Business Evaluation
Standard machine learning metrics evaluate accuracy on a per-minute basis, which is useless for alerting systems. A model that alerts on minute 1 of a 30-minute outage is a success, but standard metrics would score it poorly for missing the remaining 29 minutes.

The `business_eval` function implements an event-based evaluation:
- It groups continuous anomaly minutes into singular incident events.
- An incident is marked as Caught if the system fired at least one alert inside the incident window or within the prediction horizon leading up to it.
- False Positives are calculated as the total number of alerts minus the number of caught incidents. 

## Setup and Usage

### Prerequisites
You will need Python 3.8+ and the following libraries installed:
- `numpy`
- `pandas`
- `scikit-learn`
- `scipy`
- `matplotlib`

### Running the Pipeline
Simply execute the main script. The script will generate the data, train the model, run both the baseline and the smart decision engine, and print the comparative event-based statistics to the console. 

Finally, it will render a visual plot comparing the high-spam baseline alerts against the suppressed, accurate smart alerts.

## Future Improvements
While this system uses synthetic data with realistic precursors, it is designed to be easily adapted to real-world datasets. To transition to production, the `generate_synthetic_data` function can be replaced with a CSV loader to ingest real AWS CloudWatch metrics.
