# AI Revenue Recovery System

## Razorpay AI Buildathon 2026

### Track 03 — AI Revenue Recovery

**Recover the payments most worth recovering, not every failed payment.**

AI Revenue Recovery System is an intelligent payment-recovery engine that predicts the probability of recovering a failed payment, ranks recovery opportunities, estimates expected recovery value, and applies deterministic recovery policies to decide the appropriate intervention.

The system combines machine learning with business rules, recovery prioritization, safety guardrails, and auditability.

---

## 1. Problem

Payment failures are not equally recoverable.

A temporary server timeout may have a high probability of succeeding on a retry, while an expired card may require the customer to update their payment method.

A blanket retry strategy can therefore:

- Spend recovery capacity on low-probability cases
- Generate unnecessary retry attempts
- Increase operational and payment costs
- Create unnecessary customer friction
- Miss higher-value recovery opportunities

The objective of this project is to answer:

> Which failed payments should be prioritized for recovery?

Instead of retrying every failed payment equally, the system predicts recovery probability and ranks failed payments according to their likelihood of recovery.

---

## 2. Solution

The proposed system separates prediction from decision-making.

The machine-learning model predicts the probability of recovery.

The policy engine then uses the prediction together with payment context and deterministic business rules to select an appropriate action.

```text
Failed Payment
       |
       v
Feature Extraction
       |
       v
Global CatBoost Model
       |
       v
Recovery Probability
       |
       v
Expected Recovery Value
       |
       v
Policy Engine
       |
       +------> Retry
       |
       +------> Schedule Retry
       |
       +------> Cooldown
       |
       +------> Request Payment Method Update
       |
       +------> Stop
       |
       +------> Human Review
       |
       v
Audit Log
       |
       v
Feedback and Retraining
````

The system therefore follows five major stages:

1. Predict recovery probability
2. Prioritize recovery opportunities
3. Select a recovery action
4. Apply safety and retry policies
5. Record the decision for auditing and future learning

---

## 3. Key Idea

The central idea is to treat revenue recovery as a **prioritization problem** rather than a blanket retry problem.

For every failed payment, the system estimates:

```text
P(Recovery | payment context)
```

The probability can then be combined with the transaction amount:

```text
Expected Recovery Value
=
P(Recovery) × Transaction Amount
```

For example:

```text
Transaction Amount = ₹5,000

Predicted Recovery Probability = 0.70

Expected Recovery Value
= 0.70 × ₹5,000

= ₹3,500
```

This allows the recovery system to prioritize opportunities according to both recovery likelihood and transaction value.

---

## 4. Dataset

The benchmark dataset contains 20,000 payment records covering successful payments, failed payments, recovered payments, unrecovered payments, and escalated cases.

For the revenue-recovery modeling task, the system isolates 6,284 failed-payment records containing a defined payment decline reason.

These 6,284 records form the primary recovery-analysis population.

The dataset is synthetic and was created for experimentation and evaluation of the proposed recovery strategy.

It does not contain real Razorpay customer or payment information.

### Dataset Summary

| Metric                |  Value |
| --------------------- | -----: |
| Total payment records | 20,000 |
| Failed payments       |  6,284 |
| Training records      |  3,770 |
| Calibration records   |  1,257 |
| Test records          |  1,257 |

The decline-reason counts are:
```text
2,868
+ 1,255
+   934
+   620
+   607
---------
  6,284
```

So **6,284 records have a decline reason**.

That is:

```text
6,284 / 20,000 × 100 = 31.42%
```

So approximately **31.4% of the dataset is in the failed-payment recovery scope**.


```text
                    20,000 PAYMENT RECORDS
                            |
             +--------------+--------------+
             |                             |
             v                             v
       General records              Failed-payment
                                    recovery scope
                                          |
                                          v
                                     6,284 records
                                          |
                   +----------+-----------+-----------+----------+
                   |          |           |           |          |
                   v          v           v           v          v
             insufficient  server    card_expired  risk_hold  card_limit
               funds       timeout
```

The **6,284** comes specifically from records that have a `decline_reason`.

Therefore, our model is currently answering:

> "Among failed payments, which ones are most likely to recover?"


## 5. Failed Payment Distribution

The failed-payment recovery scope contains five major decline reasons.

| Decline Reason      | Failed Payments |
| ------------------- | --------------: |
| Insufficient Funds  |           2,868 |
| Server Timeout      |           1,255 |
| Card Expired        |             934 |
| Risk Hold           |             620 |
| Card Limit Exceeded |             607 |

These failure categories behave very differently, which makes failure context an important signal for recovery decisions.

---

## 6. Recovery Distribution

The overall observed recovery rate across the failed-payment recovery scope is:

```text
32.35%
```

Recovery behavior varies significantly by decline reason.

| Decline Reason      | Observed Recovery Rate |
| ------------------- | ---------------------: |
| Server Timeout      |                 64.06% |
| Insufficient Funds  |                 32.74% |
| Card Limit Exceeded |                 32.13% |
| Risk Hold           |                  8.55% |
| Card Expired        |                  4.50% |

This difference demonstrates why a single recovery strategy is not suitable for every payment failure.

For example, a server timeout has a substantially higher observed recovery rate than an expired card.

---

## 7. Machine Learning Model

The primary model is a global CatBoost classifier.

The model is trained across all failed-payment types and predicts the probability that a failed payment will eventually recover.

```text
Payment Context
       |
       v
CatBoost Classifier
       |
       v
P(Recovery)
```

The global model was selected as the primary model because it provides a unified scoring mechanism across multiple decline reasons.

### Model Performance

The global CatBoost model achieved:

```text
ROC-AUC = 0.762
```

The evaluation used a train, calibration, and test split.

```text
Training Set       = 3,770
Calibration Set    = 1,257
Test Set           = 1,257
```

---

## 8. Model Features

The model uses payment, customer-history, and temporal information.

### Payment Features

* Transaction amount
* Decline reason
* Payment method
* Issuing bank
* Card type

### Customer Features

* Customer tenure
* Customer lifetime value
* Prior failed payments
* Prior retry success rate
* Consecutive failure count
* Customer history availability

### Temporal Features

* Near-payday indicator
* Day of month
* Day of week
* Hours since failure
* Retry attempt number

These features allow the model to capture both payment-level and behavioral recovery patterns.

---

## 9. Capacity-Constrained Prioritization

A high-performing recovery system should not only predict recovery.

It should determine **which payments to act on when recovery capacity is limited**.

The model ranks failed payments according to predicted recovery probability.

The system can then select the highest-ranked portion of the failed-payment population.

This creates a capacity-constrained recovery strategy.

### Observed Recovery Rate

| Recovery Capacity | Observed Recovery Rate |
| ----------------- | ---------------------: |
| Overall Baseline  |                 32.35% |
| Top 5%            |                 59.68% |
| Top 10%           |                 56.80% |
| Top 20%           |                 51.79% |
| Top 30%           |                 48.28% |
| Top 50%           |                 42.36% |

The results show that the highest-scored payments contain a substantially higher concentration of recoverable transactions.

---

## 10. Main Result

The selected top-20% operating point produced:

```text
Overall Baseline Recovery Rate
32.35%

Top-20% Recovery Rate
51.79%

Relative Lift
+60.0%
```

The relative lift is calculated as:

```text
Relative Lift
=
(Top-20% Recovery Rate - Baseline Recovery Rate)
/
Baseline Recovery Rate
× 100
```

Using the observed values:

```text
(51.79 - 32.35) / 32.35 × 100
≈ 60.0%
```

This means the top-20% ML-ranked segment had a recovery rate approximately 60% higher than the overall baseline in the offline evaluation.

---

## 11. Observed Recovery Value

The top-20% ML-ranked evaluation subset contained:

```text
₹337,263.42
```

in observed recovered transaction amount.

Observed recovered amount by ML capacity:

| ML Capacity | Observed Recovered Amount |
| ----------- | ------------------------: |
| Top 5%      |               ₹123,492.23 |
| Top 10%     |               ₹214,532.35 |
| Top 20%     |               ₹337,263.42 |
| Top 30%     |               ₹424,800.47 |
| Top 50%     |               ₹531,691.30 |

These values represent observed recovered transaction amounts within the corresponding evaluation subsets.

They should not be interpreted as revenue actually recovered by a production Razorpay system.

---

## 12. Why Failure Reason Matters

The model identifies `decline_reason` as its strongest feature.

This is consistent with the large difference in observed recovery rates across failure types.

For example:

```text
Server Timeout
64.06% recovery

Insufficient Funds
32.74% recovery

Card Expired
4.50% recovery
```

This suggests that the recovery mechanism should not treat all payment failures identically.

Different failure reasons require different recovery strategies.

---

## 13. Explainability

The global CatBoost feature importance results are:

| Feature                   | Importance |
| ------------------------- | ---------: |
| Decline Reason            |     82.69% |
| Near Payday               |      7.79% |
| Prior Retry Success Rate  |      2.31% |
| Day of Month              |      2.27% |
| Hours Since Failure       |      1.29% |
| Retry Attempt Number      |      0.99% |
| Consecutive Failure Count |      0.64% |
| Has History               |      0.58% |
| Amount                    |      0.42% |
| Customer Tenure           |      0.38% |
| Customer LTV              |      0.35% |
| Prior Failed Payments     |      0.20% |
| Payment Method            |      0.04% |
| Issuing Bank              |      0.04% |
| Card Type                 |      0.00% |
| Day of Week               |      0.00% |

The high importance of decline reason is expected because recovery rates differ substantially across failure categories.

The second strongest signal is the near-payday indicator.

---

## 14. Payday Effect

Timing is particularly important for insufficient-funds failures.

Observed recovery rates in the insufficient-funds evaluation set were:

| Timing          | Recovery Rate |
| --------------- | ------------: |
| Near Payday     |        44.72% |
| Not Near Payday |        24.93% |

This suggests that a failed insufficient-funds payment may become more recoverable when a customer is closer to receiving income.

This signal can therefore be used by the policy engine to schedule a retry rather than immediately retrying the payment.

Example:

```text
Insufficient Funds
        +
Near Payday
        +
High Recovery Probability
        |
        v
Schedule Retry After Payday
```

---

## 15. Recovery Policy Engine

The machine-learning model does not directly decide whether a payment should be retried.

Instead:

```text
Machine Learning
        |
        v
Predict Recovery Probability
        |
        v
Policy Engine
        |
        v
Recovery Action
```

The policy engine applies deterministic business rules and safety constraints.

Example decision logic:

```text
IF retry_attempt >= maximum_retry_attempts
    THEN STOP_RETRY

IF decline_reason == card_expired
    THEN REQUEST_PAYMENT_METHOD_UPDATE

IF decline_reason == server_timeout
    THEN COOLDOWN_THEN_RETRY

IF decline_reason == insufficient_funds
AND near_payday == true
AND recovery_probability >= threshold
    THEN SCHEDULE_RETRY_AFTER_PAYDAY

IF recovery_probability >= high_threshold
    THEN PRIORITIZE_RETRY

OTHERWISE
    THEN REVIEW_OR_DEPRIORITIZE
```

This design keeps business rules deterministic while allowing machine learning to provide recovery intelligence.

---

## 16. Recovery Actions

The system can produce several recovery actions.

### Retry

Used when the predicted recovery probability is sufficiently high and the payment is eligible for another attempt.

### Schedule Retry

Used when timing information suggests that waiting may increase the chance of recovery.

A typical example is an insufficient-funds failure near payday.

### Cooldown

Used when another attempt may be appropriate but should not happen immediately.

### Payment Method Update

Used for failures such as expired cards where retrying the same payment instrument is unlikely to succeed.

### Stop

Used when a payment reaches a retry limit or a terminal condition is reached.

### Human Review

Used for cases requiring additional judgment or where automated intervention is not appropriate.

---

## 17. Safety and Guardrails

Automated revenue recovery requires strict controls.

The system therefore separates prediction from execution and places deterministic guardrails around recovery actions.

### Retry Limits

A maximum retry count prevents unlimited payment attempts.

### Cooldown Periods

Recovery attempts can be separated by configurable waiting periods.

### Terminal Conditions

Certain conditions can immediately prevent additional automated retries.

Examples include:

```text
Card Expired
Risk Hold
Maximum Retry Attempts Reached
Customer Opt-Out
```

### Human-in-the-Loop

Low-confidence, high-value, or policy-sensitive cases can be routed for manual review.

---

## 18. Auditability

Every recovery decision should be traceable.

The audit record can contain:

```text
Payment ID
Predicted Recovery Probability
Transaction Amount
Expected Recovery Value
Selected Action
Policy Reason
Timestamp
```

Example:

```json
{
  "payment_id": "PAY_001",
  "probability": 0.73,
  "amount": 5000,
  "expected_recovery": 3650,
  "action": "SCHEDULE_RETRY_AFTER_PAYDAY",
  "reason": "high_probability_near_payday",
  "timestamp": "2026-08-30T12:30:00"
}
```

This makes recovery decisions explainable and auditable.

---

## 19. Feedback Loop

Recovery outcomes can be used to continuously improve the model.

```text
Prediction
     |
     v
Recovery Action
     |
     v
Payment Outcome
     |
     v
Recovered / Not Recovered
     |
     v
Training Dataset
     |
     v
Model Retraining
```

This creates a closed-loop system where actual payment outcomes can be used to improve future recovery predictions.

---

## 20. Specialist Model Experiment

A separate CatBoost model was evaluated specifically on insufficient-funds failures.

The specialist experiment used:

```text
Insufficient-Funds Cases = 2,868

Specialist Test Cases = 574

ROC-AUC = 0.6132
```

Its top-20% targeting result was:

```text
Recovery Rate = 43.86%
```

The specialist model also identified meaningful behavioral signals.

Its strongest features included:

| Feature                   | Importance |
| ------------------------- | ---------: |
| Near Payday               |     63.53% |
| Prior Retry Success Rate  |     10.16% |
| Day of Month              |      6.31% |
| Consecutive Failure Count |      4.68% |
| Retry Attempt Number      |      4.10% |

The specialist model is treated as an experimental analysis rather than the primary production model.

The global CatBoost model remains the primary model because it provides a unified recovery-ranking system across all decline reasons.

---

## 21. Global vs Specialist Targeting

The observed top-20% targeting results were:

| Strategy                              | Recovery Rate |
| ------------------------------------- | ------------: |
| Overall Baseline                      |        32.35% |
| Global CatBoost Top 20%               |        51.79% |
| Insufficient-Funds Specialist Top 20% |        43.86% |

These results come from different evaluation populations and therefore should not be interpreted as a controlled head-to-head benchmark.

The comparison is included to document the model-selection process.

---

## 22. Repository Structure

```text
AI_Revenue-Recovery-System/
|
├── architecture/
|   └── architecture.png
|
├── data/
|   └── README.md
|
├── graphs/
|   ├── 01_ml_capacity_recovery.png
|   ├── 02_recovery_lift.png
|   ├── 03_recovered_amount.png
|   ├── 04_recovery_by_reason.png
|   ├── 05_payday_effect.png
|   ├── 06_feature_importance.png
|   ├── 07_global_vs_specialist.png
|   ├── 08_retry_attempt.png
|   ├── 09_retry_history.png
|   └── 10_specialist_score_groups.png
|
├── notebook/
|   └── revenue_recovery.ipynb
|
├── src/
|   ├── preprocessing.py
|   ├── train.py
|   ├── predict.py
|   ├── policy_engine.py
|   ├── recovery_engine.py
|   └── audit.py
|
├── app.py
├── requirements.txt
└── README.md
```

---

## 23. Graphs

The `graphs/` directory contains the main evaluation visualizations.

### Recovery Performance

```text
01_ml_capacity_recovery.png
02_recovery_lift.png
03_recovered_amount.png
```

These visualize the relationship between ML prioritization capacity, recovery rate, relative lift, and observed recovered amount.

### Failure Analysis

```text
04_recovery_by_reason.png
05_payday_effect.png
```

These show how recovery varies by failure type and timing.

### Explainability

```text
06_feature_importance.png
```

This visualizes the global CatBoost feature importance.

### Model Experiments

```text
07_global_vs_specialist.png
08_retry_attempt.png
09_retry_history.png
10_specialist_score_groups.png
```

These visualize the specialist experiment and behavioral recovery patterns.

---

## 24. Technology Stack

The project uses:

* Python
* Pandas
* NumPy
* Scikit-learn
* CatBoost
* Matplotlib
* Streamlit
* JSON-based audit logging

The machine-learning pipeline is implemented in Python, while the recovery policy layer provides deterministic decision logic on top of model predictions.

---

## 25. Running the Project

Clone the repository:

```bash
git clone <repository-url>
```

Move into the project directory:

```bash
cd AI_Revenue-Recovery-System
```

Install dependencies:

```bash
pip install -r requirements.txt
```

Train the model:

```bash
python src/train.py
```

Generate predictions:

```bash
python src/predict.py
```

Run the demonstration application:

```bash
streamlit run app.py
```

---

## 26. Limitations

This project is an offline research and prototype system.

### Synthetic Dataset

The benchmark dataset is synthetic.

Results therefore do not represent the performance of the system on real Razorpay production traffic.

### Offline Evaluation

The recovery rates and monetary values reported in this repository are based on offline evaluation data.

They should not be interpreted as guaranteed production revenue uplift.

### Production Integration

The current project does not claim to execute real Razorpay payment retries.

Actual payment-provider integration would require production APIs, authentication, compliance controls, monitoring, and additional safety validation.

### Probability Calibration

Predicted probabilities require additional calibration and validation before they should be interpreted as literal production probabilities.

### Cost Modeling

A production deployment should incorporate actual payment, communication, operational, and risk costs into the intervention decision.

---

## 27. Future Work

### Real Payment Data

Evaluate the model using anonymized production payment outcomes.

### Cost-Aware Recovery

Extend the expected-value calculation to:

```text
Expected Net Recovery
=
P(Recovery) × Transaction Amount
− Intervention Cost
− Communication Cost
− Expected Risk Cost
```

### Dynamic Retry Scheduling

Learn the optimal retry time using:

* Payday timing
* Customer payment behavior
* Failure reason
* Time since failure
* Retry history

### Contextual Bandits

Use contextual bandits to learn which recovery intervention works best for different customer and payment contexts.

### Reinforcement Learning

A future system could learn long-term recovery policies while respecting hard business and safety constraints.

### Production Integration

Integrate the policy engine with payment-provider APIs after appropriate validation and safety review.

### Monitoring

Monitor:

* Recovery rate
* Revenue recovered
* Retry success rate
* Customer friction
* False-positive interventions
* Model drift
* Policy violations

---

## 28. Key Results

The primary global model produced the following benchmark results:

| Metric                               |      Result |
| ------------------------------------ | ----------: |
| Failed-payment scope                 |       6,284 |
| Test set                             |       1,257 |
| Global CatBoost ROC-AUC              |       0.762 |
| Overall recovery rate                |      32.35% |
| Top-20% recovery rate                |      51.79% |
| Relative lift at top 20%             |      +60.0% |
| Observed recovered amount in top 20% | ₹337,263.42 |

---

## 29. Core Insight

The most important insight from the project is:

> Revenue recovery should not be a blanket retry problem.

A recovery system should first estimate which failed payments are most likely to recover, then prioritize those opportunities according to available recovery capacity and economic value.

The final decision should remain governed by deterministic policies and safety constraints.

```text
PREDICT
   |
   v
PRIORITIZE
   |
   v
RECOVER
   |
   v
AUDIT
   |
   v
LEARN
```

---

## 30. Project Status

The current implementation demonstrates:

* Failed-payment recovery analysis
* Global CatBoost recovery prediction
* Capacity-constrained prioritization
* Recovery-rate evaluation
* Recovery-value analysis
* Failure-reason analysis
* Payday analysis
* Feature importance
* Specialist model experimentation
* Deterministic recovery policies
* Auditability design

The next stage is to connect the trained model to the recovery-policy application and demonstrate the complete prediction-to-decision workflow.

---

## Disclaimer

This project was developed as an experimental prototype for the Razorpay AI Buildathon 2026.

The evaluation uses a synthetic benchmark dataset and offline results.

The project does not claim access to Razorpay's production payment systems, customer data, or payment execution infrastructure.

```

### One change from your current GitHub

Your README can now be the **main landing page** of the project. After replacing it, the next thing we should do is **fix the repository structure**, especially `src/`, `notebook/`, and `architecture/`, and make sure the README links to the actual files rather than describing files that don't exist yet.
```
