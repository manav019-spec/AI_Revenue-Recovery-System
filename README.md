# AI Revenue Recovery System
Razorpay AI Buildathon 2026 — Track 03: AI Revenue Recovery
> **Recover the payments most worth recovering — not every failed payment.**

An AI-powered revenue recovery system that predicts the probability of recovering a failed payment, prioritizes high-value recovery opportunities, applies deterministic recovery policies, and maintains an auditable decision trail.

---

## 🚀 Overview

Payment failures do not all have the same probability of recovery.

A temporary server timeout may be highly recoverable through a retry, while an expired card may require the customer to update their payment method.

A naive recovery system may simply retry every failed payment.

This creates three problems:

- Unnecessary retry attempts
- Increased operational/payment costs
- Poor customer experience

This project introduces an AI-driven recovery engine that answers:

> **"Which failed payments should we attempt to recover first?"**

The system combines:

**Machine Learning + Expected Recovery Value + Policy Rules + Auditability**

---

# 🎯 Problem

For every failed payment, the system needs to determine:

1. How likely is this payment to recover?
2. How valuable is the opportunity?
3. What recovery action should be taken?
4. When should the payment be retried?
5. When should the system stop retrying?
6. When should the case be escalated for human review?

Instead of treating every failed payment equally, the system ranks failed payments according to their predicted recovery probability.

---

# 💡 Solution

The system follows a four-stage process:

```text
FAILED PAYMENT
      │
      ▼
FEATURE EXTRACTION
      │
      ▼
GLOBAL CATBOOST MODEL
      │
      ▼
P(RECOVERY)
      │
      ▼
EXPECTED RECOVERY VALUE
      │
      ▼
POLICY ENGINE
      │
      ├── RETRY
      ├── SCHEDULE RETRY
      ├── REQUEST PAYMENT METHOD UPDATE
      ├── COOLDOWN
      ├── STOP
      └── HUMAN REVIEW
      │
      ▼
AUDIT LOG
      │
      ▼
FEEDBACK / RETRAINING
🧠 Machine Learning Model

The primary model is a CatBoost classifier trained on failed-payment data.

The model predicts:

P(recovery | payment + customer + timing context)

The prediction is then used to rank failed payments.

Features

The model considers:

Decline reason
Near-payday indicator
Prior retry success rate
Day of month
Hours since failure
Retry attempt number
Consecutive failure count
Customer history
Transaction amount
Customer tenure
Customer lifetime value
Prior failed payments
Payment method
Issuing bank
Card type
Day of week
📊 Dataset

The benchmark dataset contains:

20,000 total payment records
6,284 failed payments in the recovery scope

Failed-payment decline reasons include:

Decline Reason	Count
Insufficient Funds	2,868
Server Timeout	1,255
Card Expired	934
Risk Hold	620
Card Limit Exceeded	607

The dataset is a synthetic benchmark dataset created for evaluating the recovery strategy.

Therefore, the results should be interpreted as offline benchmark results rather than production Razorpay revenue.

📈 Recovery Distribution

Across the failed-payment recovery scope, the overall observed recovery rate is:

32.35%

Recovery behavior varies significantly by failure reason.

Decline Reason	Recovery Rate
Server Timeout	64.06%
Insufficient Funds	32.74%
Card Limit Exceeded	32.13%
Risk Hold	8.55%
Card Expired	4.50%

This demonstrates why a single retry strategy is not appropriate for every failure type.

🤖 Model Performance

The global CatBoost model achieved:

ROC-AUC = 0.762

The evaluation used a train/calibration/test split.

Train:       3,770
Calibration: 1,257
Test:        1,257
🎯 Capacity-Constrained Recovery

The most important idea in this project is prioritization.

Instead of attempting recovery on every failed payment, the system ranks payments by predicted recovery probability.

Observed recovery rate
Strategy	Recovery Rate
Overall Baseline	32.35%
Top 5%	59.68%
Top 10%	56.80%
Top 20%	51.79%
Top 30%	48.28%
Top 50%	42.36%

The strongest practical operating point demonstrated in the benchmark is the top-20% capacity segment.

Result
Baseline Recovery Rate = 32.35%

Top-20% Recovery Rate = 51.79%

Relative Lift = +60.0%

This means the highest-scored 20% of failed payments contained a substantially higher concentration of recoverable transactions than the overall failed-payment population.

💰 Observed Recovery Value

The top-20% ML-ranked evaluation subset contained:

₹337,263.42

of observed recovered transaction amount.

Other capacity levels contained:

ML Capacity	Observed Recovered Amount
Top 5%	₹123,492.23
Top 10%	₹214,532.35
Top 20%	₹337,263.42
Top 30%	₹424,800.47
Top 50%	₹531,691.30

These are observed recovered amounts in the evaluation subsets. They should not be interpreted as money directly recovered by a deployed production system.

🔍 Explainability

The global model identifies the following features as the strongest signals:

Feature	Importance
Decline Reason	82.69%
Near Payday	7.79%
Prior Retry Success Rate	2.31%
Day of Month	2.27%
Hours Since Failure	1.29%
Retry Attempt Number	0.99%

The dominance of decline_reason is consistent with the large differences in recovery rates between failure types.

For example:

Server Timeout     → 64.06%
Insufficient Funds → 32.74%
Card Expired       → 4.50%

The model therefore first identifies fundamentally different recovery regimes and then uses behavioral and temporal signals for prioritization.

📅 Payday Signal

For insufficient-funds failures:

Near payday     → 44.72%
Not near payday → 24.93%

This demonstrates that timing is an important recovery signal.

A customer who fails due to insufficient funds immediately before receiving income may be a stronger candidate for a scheduled retry than the same customer several days away from payday.

🧪 Specialist Model Experiment

A separate CatBoost model was tested specifically for insufficient-funds failures.

Specialist model
Test cases: 574
ROC-AUC:    0.6132

Top-20% targeting achieved:

43.86% recovery

The specialist model showed that additional within-reason modeling can identify useful patterns, especially around payday timing and retry history.

However, the global CatBoost model is selected as the primary model because it provides a unified recovery-ranking system across all decline reasons.

The specialist model is therefore treated as a research/ablation experiment rather than the production model.

⚙️ Recovery Policy Engine

Machine learning predicts recovery probability.

It does NOT independently decide whether a payment should be retried.

The final decision is handled by a deterministic policy engine.

Example:

IF retry_attempt >= 3
    → STOP_RETRY

IF card_expired
    → REQUEST_PAYMENT_METHOD_UPDATE

IF server_timeout
    → COOLDOWN_THEN_RETRY

IF insufficient_funds AND near_payday
    AND probability >= threshold
    → SCHEDULE_RETRY_AFTER_PAYDAY

IF probability is high
    → PRIORITIZE_RETRY

OTHERWISE
    → REVIEW / DEPRIORITIZE

This separation provides:

Predictive intelligence
Deterministic business rules
Retry limits
Compliance guardrails
Explainable decisions
💵 Expected Recovery Value

The system converts probability into an economic prioritization signal.

Expected Recovery Value
=
P(Recovery) × Transaction Amount

Example:

Transaction Amount = ₹5,000

Predicted Recovery Probability = 0.70

Expected Recovery Value
= 0.70 × ₹5,000
= ₹3,500

A future version can extend this to:

Expected Net Recovery
=
P(Recovery) × Amount
− Intervention Cost
− Expected Penalties

This allows the recovery engine to optimize for economic value rather than probability alone.

🛡️ Safety & Guardrails

The recovery engine is designed with deterministic controls.

Retry limits

The system can stop recovery attempts after a configurable maximum number of retries.

Cooldown periods

Repeated attempts can be separated by cooldown intervals.

Terminal conditions

Examples:

Card expired
Risk hold
Maximum retry attempts reached
Customer explicitly opts out

can prevent additional automated retries.

Human-in-the-loop

Low-confidence or high-risk cases can be routed for manual review.

🧾 Auditability

Every recovery decision can be logged with:

Payment ID
Predicted probability
Transaction amount
Expected recovery value
Selected action
Policy reason
Timestamp

Example:

{
  "payment_id": "PAY_001",
  "probability": 0.73,
  "amount": 5000,
  "expected_recovery": 3650,
  "action": "SCHEDULE_RETRY_AFTER_PAYDAY",
  "reason": "high_probability_near_payday"
}

This creates an auditable trail of why the system made each recovery decision.

🔄 Feedback Loop

Recovery outcomes can be fed back into the training pipeline.

Prediction
    ↓
Recovery Action
    ↓
Payment Outcome
    ↓
Success / Failure
    ↓
Training Dataset
    ↓
Model Retraining

This allows the system to adapt as payment behavior changes.

📊 Visual Results

The repository contains the following evaluation graphs:

Recovery performance
01_ml_capacity_recovery.png
02_recovery_lift.png
03_recovered_amount.png
Failure analysis
04_recovery_by_reason.png
05_payday_effect.png
Explainability
06_feature_importance.png
Model experiments
07_global_vs_specialist.png
08_retry_attempt.png
09_retry_history.png
10_specialist_score_groups.png
🏗️ Project Structure
AI_Revenue-Recovery-System/
│
├── architecture/
│   └── architecture.png
│
├── data/
│   └── README.md
│
├── graphs/
│   ├── 01_ml_capacity_recovery.png
│   ├── 02_recovery_lift.png
│   ├── 03_recovered_amount.png
│   ├── 04_recovery_by_reason.png
│   ├── 05_payday_effect.png
│   ├── 06_feature_importance.png
│   ├── 07_global_vs_specialist.png
│   ├── 08_retry_attempt.png
│   ├── 09_retry_history.png
│   └── 10_specialist_score_groups.png
│
├── notebook/
│   └── revenue_recovery.ipynb
│
├── src/
│   ├── preprocessing.py
│   ├── train.py
│   ├── predict.py
│   ├── policy_engine.py
│   ├── recovery_engine.py
│   └── audit.py
│
├── app.py
├── requirements.txt
└── README.md
🚀 Running the Project
1. Clone the repository
git clone <repository-url>
cd AI_Revenue-Recovery-System
2. Install dependencies
pip install -r requirements.txt
3. Train the model
python src/train.py
4. Generate predictions
python src/predict.py
5. Run the recovery demo
streamlit run app.py
🧪 Limitations

This project is an offline benchmark/prototype.

Synthetic data

The dataset is synthetic and does not represent real Razorpay customer data.

Offline evaluation

The reported recovery rates are observed results on evaluation data and should not be interpreted as guaranteed production revenue uplift.

Model calibration

Predicted probabilities require further calibration and validation before being used as literal production probabilities.

Production integration

Actual payment execution, gateway integration, customer messaging, and payment-provider APIs are outside the current prototype.

🔮 Future Work
1. Real payment data

Train and validate on anonymized production payment outcomes.

2. Cost-aware optimization

Optimize:

Expected Net Recovery
=
Expected Recovery
− Retry Cost
− Communication Cost
− Risk Cost
3. Contextual bandits / reinforcement learning

Learn which intervention works best for each customer context.

4. Dynamic retry scheduling

Optimize retry timing using:

Payday
Historical payment behavior
Failure reason
Time since failure
Customer behavior
5. Real payment gateway integration

Connect the policy engine to payment retry APIs.

6. Monitoring

Track:

Recovery rate
Revenue recovered
Retry success rate
Customer friction
False-positive interventions
Model drift
🏆 Key Takeaway

The core insight of this project is:

Revenue recovery should be treated as a prioritization problem, not a blanket retry problem.

The global CatBoost model achieved:

ROC-AUC                  0.762

Overall recovery        32.35%

Top-20% recovery        51.79%

Relative lift            +60.0%

Observed recovered
amount in top-20%       ₹337,263.42

The system combines these predictions with deterministic recovery policies, economic prioritization, safety guardrails, and audit logging.

PREDICT
   ↓
PRIORITIZE
   ↓
RECOVER
   ↓
AUDIT
   ↓
LEARN
Disclaimer

This project is an experimental AI revenue-recovery prototype developed for the Razorpay AI Buildathon.

It uses a synthetic benchmark dataset and offline evaluation. It does not claim access to or execution against Razorpay's production payment systems.
