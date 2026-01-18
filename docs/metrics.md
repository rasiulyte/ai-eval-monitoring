# Metrics: What We Measure and Why

## Overview

This document defines the metrics that answer: **"Is our AI assistant's quality holding, and if not, where is it failing?"**

Metrics are organized into three categories:
- **Quality metrics** — Is the assistant responding well?
- **Operational metrics** — Is the evaluation system working?
- **Calibration metrics** — Can we trust the autograder?

---

## Quality Metrics

These measure the AI assistant's response quality.

### Overall Quality Score

| Attribute | Value |
|-----------|-------|
| **Definition** | Mean score across all 5 dimensions (correctness, completeness, conciseness, naturalness, safety) |
| **Source** | Tier 2 autograder |
| **Range** | 1.0 – 5.0 |
| **Good** | ≥ 4.2 |
| **Alert threshold** | < 4.0 for 2 consecutive hours |
| **Granularity** | Hourly, daily |

**Why it matters:** Single number to answer "how are we doing overall?"

---

### Quality Score by Dimension

| Attribute | Value |
|-----------|-------|
| **Definition** | Mean score for each dimension separately |
| **Source** | Tier 2 autograder |
| **Dimensions** | Correctness, Completeness, Conciseness, Naturalness, Safety |
| **Good** | All dimensions ≥ 4.0 |
| **Alert threshold** | Any dimension < 3.5 for 2 consecutive hours |

**Why it matters:** Pinpoints which aspect is degrading. Overall score might be fine while safety is dropping.

---

### Safety Failure Rate

| Attribute | Value |
|-----------|-------|
| **Definition** | % of responses with Safety score < 3 |
| **Source** | Tier 2 autograder |
| **Good** | < 0.1% |
| **Alert threshold** | > 0.5% in any hour |
| **Severity** | Critical — pages on-call immediately |

**Why it matters:** Safety failures are the highest risk. Even a small spike needs immediate attention.

---

### Failure Rate by Category

| Attribute | Value |
|-----------|-------|
| **Definition** | % of responses scoring < 3 on any dimension, grouped by query category |
| **Categories** | Factual, Task, Safety, Subjective, Math, Weather, Edge cases |
| **Source** | Tier 2 autograder + query classifier |
| **Good** | No category > 5% failure rate |
| **Alert threshold** | Any category > 10% failure rate |

**Why it matters:** A model update might break one category while others stay fine. This catches targeted regressions.

---

## Operational Metrics

These measure whether the evaluation system itself is working.

### Evaluation Throughput

| Attribute | Value |
|-----------|-------|
| **Definition** | Number of responses evaluated per hour at each tier |
| **Expected** | Tier 1: ~417K/hr, Tier 2: ~4.2K/hr, Tier 3: ~210/hr, Tier 4: ~2/hr |
| **Alert threshold** | < 50% of expected for 1 hour |

**Why it matters:** If throughput drops, we have a blind spot. Could indicate pipeline failure.

---

### Evaluation Latency

| Attribute | Value |
|-----------|-------|
| **Definition** | Time from response generated to quality score available |
| **Target** | < 1 hour for Tier 2 scores |
| **Alert threshold** | > 2 hours |

**Why it matters:** Stale scores mean we detect problems late.

---

### Tier Escalation Rate

| Attribute | Value |
|-----------|-------|
| **Definition** | % of Tier 2 evaluations that get flagged to Tier 3 |
| **Expected** | ~5% |
| **Alert threshold** | > 15% (too many flags) or < 1% (autograder may be broken) |

**Why it matters:** Sudden spike means something changed—either the assistant got worse or the autograder is misfiring.

---

## Calibration Metrics

These measure whether we can trust the autograder.

### Autograder-Human Agreement

| Attribute | Value |
|-----------|-------|
| **Definition** | Cohen's Kappa between Tier 2 scores and Tier 4 human labels |
| **Source** | Compare Tier 2 vs Tier 4 on same responses |
| **Good** | κ ≥ 0.7 |
| **Alert threshold** | κ < 0.6 (weekly check) |

**Why it matters:** If autograder drifts from human judgment, our metrics become meaningless.

---

### Autograder Bias

| Attribute | Value |
|-----------|-------|
| **Definition** | Mean signed difference (autograder score - human score) |
| **Source** | Tier 4 human reviews |
| **Good** | Between -0.2 and +0.2 |
| **Alert threshold** | Bias > +0.5 or < -0.5 |

**Why it matters:** Systematic over/under-rating hides real problems or creates false alarms.

---

### Tier 3 Overturn Rate

| Attribute | Value |
|-----------|-------|
| **Definition** | % of Tier 2 flags that Tier 3 overturns (disagrees with) |
| **Expected** | 20-40% |
| **Alert threshold** | > 60% (Tier 2 too aggressive) or < 10% (Tier 2 missing problems) |

**Why it matters:** Measures alignment between cheap and expensive autograders.

---

## Metric Summary Table

| Metric | Source | Good | Alert | Severity |
|--------|--------|------|-------|----------|
| Overall quality score | Tier 2 | ≥ 4.2 | < 4.0 (2 hrs) | High |
| Dimension scores | Tier 2 | All ≥ 4.0 | Any < 3.5 (2 hrs) | High |
| Safety failure rate | Tier 2 | < 0.1% | > 0.5% (1 hr) | Critical |
| Category failure rate | Tier 2 | All < 5% | Any > 10% | Medium |
| Evaluation throughput | Pipeline | At expected | < 50% (1 hr) | High |
| Evaluation latency | Pipeline | < 1 hr | > 2 hrs | Medium |
| Escalation rate | Tier 2→3 | ~5% | > 15% or < 1% | Medium |
| Autograder-human agreement | Tier 4 | κ ≥ 0.7 | κ < 0.6 | High |
| Autograder bias | Tier 4 | ±0.2 | > ±0.5 | Medium |
| Tier 3 overturn rate | Tier 3 | 20-40% | > 60% or < 10% | Medium |

---

## Next Steps

- [CI/CD pipelines](ci-cd-pipelines.md) — quality gates using these metrics
- [Alerting logic](alerting.md) — when and how we notify based on thresholds

