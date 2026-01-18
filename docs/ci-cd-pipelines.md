# CI/CD Pipelines: Quality Gates for AI Deployment

## Overview

Two things change in a production AI system:

1. **The AI assistant** — model updates, prompt changes
2. **The evaluation system** — autograder prompts, rubrics, thresholds

Both need CI/CD pipelines with quality gates. A bad model degrades user experience. A bad autograder hides problems or creates false alarms.

---

## Pipeline Architecture
```
┌─────────────────────────────────────────────────────────────────────┐
│                        CI/CD LAYER                                  │
├─────────────────────────────────┬───────────────────────────────────┤
│   Assistant Pipeline            │   Evaluator Pipeline              │
│                                 │                                   │
│   Code/prompt change            │   Rubric/threshold change         │
│         ↓                       │         ↓                         │
│   Run golden set eval           │   Run on labeled test set         │
│         ↓                       │         ↓                         │
│   Compare to baseline           │   Compare to human agreement      │
│         ↓                       │         ↓                         │
│   Gate: quality ≥ threshold?    │   Gate: agreement ≥ 0.7 kappa?    │
│         ↓                       │         ↓                         │
│   Shadow deployment             │   A/B test vs current autograder  │
│         ↓                       │         ↓                         │
│   Gradual rollout               │   Deploy new autograder           │
└─────────────────────────────────┴───────────────────────────────────┘
                    ↓                           ↓
              Production Assistant        Monitoring System
                    ↓                           ↓
              Response Logs ──────────────→ Tiered Evaluation
                                                ↓
                                          Quality Metrics
                                                ↓
                                   ┌────────────────────────┐
                                   │ Feedback Loop:         │
                                   │ - New failure cases    │
                                   │   added to golden set  │
                                   │ - Rubric refinements   │
                                   │   trigger eval CI/CD   │
                                   └────────────────────────┘
```

---

## Assistant Pipeline (Model/Prompt Changes)

This pipeline runs when engineers update the AI assistant—new model version, prompt changes, or system updates.

### Stage 1: Golden Set Evaluation (like BVT)

| Attribute | Value |
|-----------|-------|
| **What** | Run new model against 500+ curated test cases |
| **Test cases** | Critical scenarios: safety, core functionality, known past failures |
| **Pass criteria** | Zero safety failures, all critical cases pass |
| **Fail action** | Block deployment, notify developer |
| **Duration** | ~10 minutes |

**Example golden set categories:**

| Category | Count | Must pass |
|----------|-------|-----------|
| Safety (harmful request refusals) | 100 | 100% |
| Core functionality | 200 | 95% |
| Known past failures | 100 | 100% |
| Edge cases | 100 | 80% |

---

### Stage 2: Baseline Comparison (like Regression)

| Attribute | Value |
|-----------|-------|
| **What** | Compare new model scores to current production model |
| **Test set** | Same golden set + 1000 random production samples |
| **Pass criteria** | Quality score within 0.2 of baseline on all dimensions |
| **Fail action** | Block deployment, review regressions |
| **Duration** | ~30 minutes |

**Comparison metrics:**

| Metric | Baseline | New Model | Pass? |
|--------|----------|-----------|-------|
| Overall quality | 4.3 | 4.2 | ✓ (within 0.2) |
| Safety score | 4.8 | 4.7 | ✓ |
| Correctness | 4.2 | 3.9 | ✗ (dropped 0.3) |

If any dimension drops more than 0.2, deployment is blocked.

---

### Stage 3: Shadow Deployment (like Integration)

| Attribute | Value |
|-----------|-------|
| **What** | Run new model on 1% of live traffic without showing users |
| **How** | Duplicate requests, evaluate both responses, compare |
| **Pass criteria** | New model failure rate ≤ 1.1× current model |
| **Fail action** | Halt rollout, investigate |
| **Duration** | 2-4 hours |

**Why shadow first?** Golden set is curated. Real traffic has surprises. Shadow catches issues that tests miss.

---

### Stage 4: Gradual Rollout

| Attribute | Value |
|-----------|-------|
| **What** | Slowly shift traffic to new model with monitoring |
| **Stages** | 1% → 10% → 50% → 100% |
| **Gate at each stage** | Quality metrics hold, no alert triggers |
| **Rollback trigger** | Any critical alert, quality drop > 0.3 |
| **Duration** | 24-48 hours total |

**Rollout schedule:**

| Stage | Traffic | Duration | Rollback if |
|-------|---------|----------|-------------|
| 1% | 100K responses | 4 hours | Any safety alert |
| 10% | 1M responses | 8 hours | Quality < 4.0 |
| 50% | 5M responses | 12 hours | Quality < 4.0 |
| 100% | 10M responses | Ongoing | Quality < 4.0 |

---

## Evaluator Pipeline (Autograder Changes)

This pipeline runs when engineers update the evaluation system—autograder prompts, rubrics, or scoring thresholds.

### Stage 1: Human Agreement Check

| Attribute | Value |
|-----------|-------|
| **What** | Run new autograder on human-labeled test set |
| **Test set** | 200+ responses with ground truth human scores |
| **Pass criteria** | Cohen's Kappa ≥ 0.7 |
| **Fail action** | Block deployment, review disagreements |

---

### Stage 2: Bias Check

| Attribute | Value |
|-----------|-------|
| **What** | Compare score distributions to current autograder |
| **Pass criteria** | Mean score shift < 0.3 |
| **Fail action** | Review if intentional or problematic |

**Why this matters:** An autograder that suddenly scores everything 0.5 points higher will hide real problems.

---

### Stage 3: A/B Comparison

| Attribute | Value |
|-----------|-------|
| **What** | Run both old and new autograder on same sample |
| **Sample** | 10K production responses |
| **Pass criteria** | Disagreement rate < 10% |
| **Fail action** | Review disagreements before deploying |

---

### Stage 4: Staged Rollout

| Attribute | Value |
|-----------|-------|
| **What** | New autograder handles increasing % of evaluations |
| **Stages** | 10% → 50% → 100% |
| **Gate** | No anomalies in quality metrics |
| **Duration** | 24 hours |

---

## Feedback Loop: Continuous Improvement (Hillclimbing)

The CI/CD pipeline isn't just for deployment—it drives continuous improvement through **hillclimbing**.

Hillclimbing is an iterative methodology: detect a problem, analyze root cause, implement a fix, validate improvement, deploy. Repeat.

### Two Hillclimbing Loops

This system has two parallel hillclimbing loops:

| Loop | What improves | Triggered by | Deploys via |
|------|---------------|--------------|-------------|
| Autograder hillclimbing | Evaluation accuracy | Human-autograder disagreements | Evaluator Pipeline |
| Assistant hillclimbing | Response quality | Quality metric drops | Assistant Pipeline |

### Autograder Hillclimbing

When humans and autograder disagree, we improve the autograder:
```
Tier 4 human review finds disagreement
         ↓
Analyze: Why did autograder miss this?
   - Rubric unclear?
   - Prompt allowing misinterpretation?
   - Edge case not covered?
         ↓
Fix: Adjust prompt, rubric, or add examples
   Example: "Rate conciseness (1-5)"
   Becomes: "Rate conciseness (1-5). Note: Longer 
            responses are NOT automatically better."
         ↓
Validate: Re-run on labeled test set
   - Agreement improves (κ 0.65 → 0.75)?
   - No regression on other dimensions?
         ↓
Deploy: New autograder via Evaluator Pipeline
```

### Assistant Hillclimbing

When quality metrics drop, we improve the assistant:
```
Monitoring detects category failure spike
   Example: "math" category at 15% failure rate
         ↓
Analyze: Pull failure samples, identify pattern
   Finding: Model fails multi-step calculations
         ↓
Fix: Engineers improve model or prompt
   - Add chain-of-thought for math queries
   - Fine-tune on math examples
         ↓
Validate: Run against golden set + baseline
   - Math category improves?
   - No regression elsewhere?
         ↓
Deploy: New model via Assistant Pipeline
         ↓
Update: Add failure cases to golden set
   - Prevents future regression
```

### The Virtuous Cycle

Both loops feed each other:
```
Production Traffic
       ↓
Tiered Evaluation (Tiers 1-4)
       ↓
┌──────────────────────────────────────────┐
│           Failures Detected              │
└──────────────────┬───────────────────────┘
                   ↓
     ┌─────────────┴─────────────┐
     ↓                           ↓
Autograder wrong?           Assistant wrong?
     ↓                           ↓
Hillclimb autograder        Hillclimb assistant
     ↓                           ↓
Better evaluation ──────→ Catches more failures
                                 ↓
                          Better assistant
                                 ↓
                          Fewer failures to catch
```

**Better evaluation finds more problems. Fixing problems improves quality. Improved quality raises the bar for evaluation.**

### Hillclimbing Metrics

Track improvement over time:

| Metric | Tracks | Goal |
|--------|--------|------|
| Autograder-human agreement (κ) | Autograder hillclimbing | Increasing toward 0.8+ |
| Overall quality score | Assistant hillclimbing | Increasing toward 4.5+ |
| Golden set size | Coverage | Growing as new failures discovered |
| Time to detect regression | System maturity | Decreasing |

**Examples:**

| Failure found | Action |
|---------------|--------|
| Model gives dangerous cooking advice | Add to golden set safety cases |
| Autograder misses sarcasm | Add sarcasm examples to autograder test set |
| Rubric unclear on partial answers | Refine rubric, re-run evaluator pipeline |

---

## Pipeline Summary

| Pipeline | Trigger | Stages | Total Duration |
|----------|---------|--------|----------------|
| Assistant | Model/prompt change | Golden set → Baseline → Shadow → Rollout | 24-48 hours |
| Evaluator | Autograder/rubric change | Agreement → Bias → A/B → Rollout | 24 hours |

---

## Rollback Procedures

### Assistant Rollback

| Trigger | Action | Time to rollback |
|---------|--------|------------------|
| Critical safety alert | Automatic rollback to previous model | < 5 minutes |
| Quality drop > 0.3 | Manual review, then rollback | < 30 minutes |
| Gradual degradation | Investigate first, rollback if confirmed | < 2 hours |

### Evaluator Rollback

| Trigger | Action | Time to rollback |
|---------|--------|------------------|
| Agreement drops < 0.5 | Automatic rollback | < 5 minutes |
| Anomalous metrics | Manual review, then rollback | < 1 hour |

---

## Next Steps

- [Alerting logic](alerting.md) — when and how we notify based on pipeline and metric events