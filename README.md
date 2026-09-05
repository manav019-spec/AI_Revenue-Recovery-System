```markdown
# AI Revenue Recovery System
**Razorpay AI Buildathon 2026 — Track 03: AI Revenue Recovery**

> *Recover the payments most worth recovering — not every failed payment.*

An intelligent, machine-learning-driven revenue recovery engine designed to predict the probability of recovering failed transactions, prioritize high-value opportunities, execute deterministic policy rules, and maintain an immutable, auditable decision log.

---

## Executive Summary

Standard payment recovery implementations typically apply blanket retries to all failed payments. This naive approach leads to unnecessary operational and processing costs, increased rate limits, and poor customer experience.

The AI Revenue Recovery System reframes payment recovery as an **optimization and prioritization problem**. By combining machine learning predictions with deterministic business guardrails and economic value calculations, the engine determines which payments to attempt, when to schedule retries, and when to halt intervention.

### Key Metrics & Benchmark Highlights
* **Global Model Performance:** `0.762 ROC-AUC` using a calibrated CatBoost classifier.
* **Baseline vs. ML-Targeted Recovery:**
  * **Baseline Failure Recovery Rate:** `32.35%`
  * **Top-20% Prioritized Recovery Rate:** `51.79%`
  * **Relative Recovery Lift:** `+60.0%`
* **Economic Value Captured:** `₹337,263.42` in observed recovered value within the top-20% priority capacity on test evaluation subsets.

---

## Architecture & System Flow

```text
FAILED PAYMENT EVENT
         │
         ▼
[ Feature Extraction ]
         │
         ▼
[ CatBoost Predictive Model ] ──► P(Recovery)
         │
         ▼
[ Expected Recovery Value Engine ] ──► P(Recovery) × Transaction Amount
         │
         ▼
[ Policy Engine & Guardrails ]
         │
         ├── RETRY (Immediate execution)
         ├── SCHEDULE_RETRY (Payday / time-shifted execution)
         ├── REQUEST_PAYMENT_METHOD_UPDATE (Expired / terminal failure)
         ├── COOLDOWN (Temporary failure back-off)
         ├── HUMAN_REVIEW (High value / low confidence cases)
         └── STOP (Terminal failure state reached)
         │
         ▼
[ Audit Log & Feedback Loop ]

```

---

## Dataset Overview & Empirical Findings

The underlying benchmark dataset consists of 20,000 transaction records containing **6,284 failed payments** in scope for recovery modeling.

### Decline Reason Distribution & Observed Baseline Recovery

| Decline Reason | Total Failures | Baseline Recovery Rate | Recovery Characteristics |
| --- | --- | --- | --- |
| **Server Timeout** | 1,255 | 64.06% | High recovery via short-term retries. |
| **Insufficient Funds** | 2,868 | 32.74% | Highly sensitive to payday and timing context. |
| **Card Limit Exceeded** | 607 | 32.13% | Moderate recovery over billing cycles. |
| **Risk Hold** | 620 | 8.55% | Low automated recovery; requires intervention. |
| **Card Expired** | 934 | 4.50% | Terminal failure; requires user credentials update. |
| **Total / Overall** | **6,284** | **32.35%** | **Overall baseline recovery across all scope.** |

---

## Model Performance & Prioritization Strategy

### Capacity-Constrained Recovery Lift

By ranking failed payments by predicted recovery probability $P(\text{Recovery})$, the engine maximizes recovery yield under operational or retry capacity constraints:

| Capacity Segment | Target Recovery Rate | Observed Recovered Amount | Strategy Lift |
| --- | --- | --- | --- |
| **Top 5%** | 59.68% | ₹123,492.23 | Highest probability concentration |
| **Top 10%** | 56.80% | ₹214,532.35 | High-efficiency targeting |
| **Top 20% (Optimal)** | **51.79%** | **₹337,263.42** | **+60.0% relative lift over baseline** |
| **Top 30%** | 48.28% | ₹424,800.47 | Broad capture |
| **Top 50%** | 42.36% | ₹531,691.30 | Moderate concentration |
| **Baseline (100%)** | 32.35% | -- | Unranked, naive retries |

### Global Feature Importance Analysis

Model explainability yields the following top feature contributions:

```text
Decline Reason           : ■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■ 82.69%
Near Payday Indicator    : ■■■ 7.79%
Prior Retry Success Rate : ■ 2.31%
Day of Month             : ■ 2.27%
Hours Since Failure      : ■ 1.29%
Retry Attempt Number     : ■ 0.99%

```

### The Payday Signal Impact

Temporal features show high predictive power for specific failure types, particularly **Insufficient Funds**:

* **Insufficient Funds (Near Payday):** `44.72%` recovery rate
* **Insufficient Funds (Not Near Payday):** `24.93%` recovery rate

---

## Economic Prioritization & Policy Engine

### 1. Expected Recovery Value (ERV)

Decisions are ranked based on economic utility, balancing recovery likelihood against transaction magnitude:

$$\text{Expected Recovery Value (ERV)} = P(\text{Recovery}) \times \text{Transaction Amount}$$

*Example:* For a transaction amount of ₹5,000 with $P(\text{Recovery}) = 0.73$, the $\text{ERV} = \text{₹3,650}$.

### 2. Policy Engine Logic

The ML output is evaluated against deterministic business guardrails to select the final action:

```python
IF retry_attempt >= MAX_RETRIES:
    ACTION = "STOP"

ELIF decline_reason == "CARD_EXPIRED":
    ACTION = "REQUEST_PAYMENT_METHOD_UPDATE"

ELIF decline_reason == "SERVER_TIMEOUT":
    ACTION = "COOLDOWN_THEN_RETRY"

ELIF decline_reason == "INSUFFICIENT_FUNDS" AND near_payday AND probability >= THRESHOLD:
    ACTION = "SCHEDULE_RETRY_AFTER_PAYDAY"

ELIF probability >= HIGH_CONFIDENCE_THRESHOLD:
    ACTION = "PRIORITIZE_RETRY"

ELSE:
    ACTION = "DEPRIORITIZE_OR_HUMAN_REVIEW"

```

---

## Auditability & Logging

Every automated decision generates a structured, auditable log entry containing model outputs, feature inputs, and applied policy rationale:

```json
{
  "payment_id": "PAY_98412_2026",
  "timestamp": "2026-09-05T17:50:00Z",
  "amount": 5000.00,
  "predicted_probability": 0.73,
  "expected_recovery_value": 3650.00,
  "selected_action": "SCHEDULE_RETRY_AFTER_PAYDAY",
  "policy_rule_triggered": "high_probability_near_payday_insufficient_funds",
  "terminal_state": false
}

```

---

## Project Structure

```text
AI_Revenue-Recovery-System/
├── architecture/
│   └── architecture.png
├── data/
│   └── README.md
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
├── notebook/
│   └── revenue_recovery.ipynb
├── src/
│   ├── preprocessing.py
│   ├── train.py
│   ├── predict.py
│   ├── policy_engine.py
│   ├── recovery_engine.py
│   └── audit.py
├── app.py
├── requirements.txt
└── README.md

```

---

## Quickstart & Execution

### 1. Prerequisites & Environment Setup

Clone the repository and install required dependencies:

```bash
git clone [https://github.com/your-username/AI_Revenue-Recovery-System.git](https://github.com/your-username/AI_Revenue-Recovery-System.git)
cd AI_Revenue-Recovery-System
pip install -r requirements.txt

```

### 2. Training the Model

Train the global CatBoost classifier and output validation metrics:

```bash
python src/train.py

```

### 3. Executing the Prediction Engine

Generate recovery probabilities and expected recovery values on candidate data:

```bash
python src/predict.py

```

### 4. Running the Interactive Dashboard

Launch the Streamlit web application to inspect recovery decisions and system parameters:

```bash
streamlit run app.py

```

---

## Limitations & Disclaimer

* **Prototype Scope:** This project is an experimental prototype built for the Razorpay AI Buildathon 2026.
* **Synthetic Data:** Benchmark evaluations were conducted using synthetic datasets to replicate real-world payment failure profiles and offline behavioral metrics.
* **Integration Boundary:** Live gateway APIs, messaging networks, and production execution pipes are omitted from this core decision engine.

---

## Future Roadmap

1. **Cost-Aware Optimization Framework:** Integrating gateway retry fees and communication overhead into net value formulation:

$$\text{Expected Net Recovery} = \text{ERV} - \text{Intervention Cost} - \text{Risk Penalty}$$


2. **Contextual Bandits:** Transitioning from static ML classification to dynamic multi-armed bandits for adaptive intervention selection.
3. **Dynamic Gateway Routing:** Coupling recovery attempts with dynamic payment gateway routing to maximize success based on live processor status.

```

```
