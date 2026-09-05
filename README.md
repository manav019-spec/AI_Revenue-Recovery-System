# AI_Revenue-Recovery-System
Razorpay AI Buildathon 2026 — Track 03: AI Revenue Recovery

# AI Revenue Recovery Engine

### ML-driven prioritization and automated recovery of failed payments

An AI-powered revenue recovery system that predicts the probability of recovering failed payments, prioritizes high-value opportunities, recommends recovery actions, and enforces deterministic recovery policies with a complete audit trail.

## Problem

Not every failed payment has the same probability of recovery.

Traditional recovery systems often rely on fixed retry schedules and static rules. This can lead to inefficient retries, poor prioritization, and missed recoverable revenue.

Our system learns recovery patterns from payment, customer, historical and temporal signals and ranks failed payments according to their expected recovery value.

## Solution

```text
Failed Payment
      ↓
Feature Extraction
      ↓
CatBoost Recovery Scorer
      ↓
P(recovery)
      ↓
Expected Recovery Value
      ↓
Policy Engine
      ↓
Recovery Action
      ↓
Outcome + Audit Log
      ↓
Feedback Loop
```

## Machine Learning

The primary model is a CatBoost classifier trained on mixed numerical and categorical payment/customer features.

The model predicts:

```text
P(recovery | customer + payment context)
```

The scoring layer then estimates:

```text
Expected Recovery Value =
P(recovery) × transaction amount
```

Hard recovery constraints such as retry limits, cooldowns and terminal failure handling remain deterministic in the policy layer.

## Results

Overall model discrimination:

**Test AUC ≈ 0.76**

For insufficient-funds failures:

| Strategy                 | Recovery Rate |
| ------------------------ | ------------: |
| Baseline                 |        32.12% |
| Near-payday heuristic    |        44.72% |
| Prior retry success ≥50% |        37.50% |
| **ML top 20%**           |    **52.29%** |

The ML-ranked top 20% achieved a **62.82% relative lift in observed recovery rate** over the overall insufficient-funds baseline in offline evaluation.

## Model Comparison

We also evaluated a segment-specific insufficient-funds CatBoost model.

| Model                |            Result |
| -------------------- | ----------------: |
| Payday-only baseline |         AUC 0.588 |
| Payday + history     |         AUC 0.607 |
| Specialist CatBoost  |         AUC 0.613 |
| **Global CatBoost**  | **≈0.76 overall** |

The specialist model demonstrated meaningful ranking ability but did not outperform the global model on the primary targeting metric, so the global model was selected for the recovery scoring layer.

## Key Insight

The model discovered a strong temporal recovery signal for insufficient-funds failures. In the evaluated test set:

* Not near payday: 24.93% recovery
* Near payday: 44.72% recovery

This supports adaptive recovery timing rather than blindly retrying every failed payment immediately.

## Recovery Policy

The ML model does not override deterministic recovery constraints.

Example policies:

* Temporary infrastructure failures → cooldown and retry
* Insufficient funds + high recovery score → prioritize retry/scheduled retry
* Expired payment method → request payment-method update
* Retry limit reached → stop or escalate
* Low-value/low-probability opportunities → deprioritize

## Explainability

Each recovery decision can expose:

* recovery probability
* transaction amount
* expected recovery value
* contributing features
* selected action
* policy decision
* model version
* execution result

## Project Structure

```text
src/
├── preprocessing.py
├── model.py
├── scoring.py
├── policy_engine.py
├── recovery_engine.py
└── audit.py

notebooks/
├── 01_data_analysis.ipynb
├── 02_global_catboost.ipynb
├── 03_model_evaluation.ipynb
└── 04_specialist_model_experiment.ipynb
```

## Limitations

The reported results are based on offline evaluation. They demonstrate targeting potential rather than guaranteed production revenue uplift.

Future work includes action-level modeling, online experimentation, adaptive retry scheduling, causal/uplift modeling and expected net recovery value optimization.

## Team

Built for the Razorpay AI Buildathon 2026 — AI Revenue Recovery.
