# Alerting: When and How We Notify

## Overview

Alerts answer: **"Something is wrong—who needs to know, and how urgently?"**

Good alerting is:
- **Actionable** — every alert has a clear response
- **Prioritized** — critical issues wake people up, minor issues wait
- **Not noisy** — too many alerts = people ignore them

---

## Alert Severity Levels

| Severity | Response Time | Who Gets Notified | Example |
|----------|---------------|-------------------|---------|
| **Critical** | < 15 minutes | On-call engineer (page) | Safety failure rate > 0.5% |
| **High** | < 2 hours | On-call engineer (Slack) | Quality score < 4.0 |
| **Medium** | < 24 hours | Team channel (Slack) | Autograder agreement dropping |
| **Low** | Next business day | Weekly report | Minor threshold breach |

---

## Alert Definitions

### Critical Alerts (Page Immediately)

These indicate potential user harm or major system failure.

#### CRIT-01: Safety Failure Spike

| Attribute | Value |
|-----------|-------|
| **Condition** | Safety failure rate > 0.5% for 1 hour |
| **Normal** | < 0.1% |
| **Action** | Page on-call, investigate immediately |
| **Possible causes** | Model regression, prompt injection attack, new harmful query pattern |
| **Runbook** | Check recent deployments, review flagged responses, consider rollback |

#### CRIT-02: Evaluation Pipeline Down

| Attribute | Value |
|-----------|-------|
| **Condition** | Tier 2 throughput = 0 for 30 minutes |
| **Normal** | ~4,200 evaluations/hour |
| **Action** | Page on-call, restore pipeline |
| **Possible causes** | API outage, queue failure, credential expiry |
| **Runbook** | Check API status, verify credentials, restart workers |

#### CRIT-03: Rollout Quality Crash

| Attribute | Value |
|-----------|-------|
| **Condition** | Quality drops > 0.5 within 1 hour during rollout |
| **Normal** | Stable or improving |
| **Action** | Auto-rollback triggered, page on-call to investigate |
| **Possible causes** | Bad model deployment, prompt error |
| **Runbook** | Verify rollback succeeded, review deployment diff |

---

### High Alerts (Respond Within 2 Hours)

These indicate quality degradation that needs attention but isn't immediately harmful.

#### HIGH-01: Quality Score Drop

| Attribute | Value |
|-----------|-------|
| **Condition** | Overall quality < 4.0 for 2 consecutive hours |
| **Normal** | ≥ 4.2 |
| **Action** | Notify on-call via Slack, investigate |
| **Possible causes** | Model drift, traffic pattern change, upstream data issue |
| **Runbook** | Check dimension breakdown, identify affected categories, review samples |

#### HIGH-02: Dimension-Specific Drop

| Attribute | Value |
|-----------|-------|
| **Condition** | Any dimension < 3.5 for 2 consecutive hours |
| **Normal** | All dimensions ≥ 4.0 |
| **Action** | Notify on-call, investigate specific dimension |
| **Possible causes** | Model regression on specific capability |
| **Runbook** | Pull samples for that dimension, compare to baseline |

#### HIGH-03: Evaluation Latency

| Attribute | Value |
|-----------|-------|
| **Condition** | Time from response to score > 2 hours |
| **Normal** | < 1 hour |
| **Action** | Notify on-call, investigate pipeline |
| **Possible causes** | Queue backup, slow API responses, worker issues |
| **Runbook** | Check queue depth, API latency, worker health |

#### HIGH-04: Autograder Agreement Drop

| Attribute | Value |
|-----------|-------|
| **Condition** | Cohen's Kappa < 0.6 (weekly check) |
| **Normal** | ≥ 0.7 |
| **Action** | Notify team, schedule autograder review |
| **Possible causes** | Autograder drift, rubric ambiguity, human labeler inconsistency |
| **Runbook** | Review disagreement samples, check for rubric updates needed |

---

### Medium Alerts (Respond Within 24 Hours)

These indicate trends that need attention but aren't urgent.

#### MED-01: Category Failure Spike

| Attribute | Value |
|-----------|-------|
| **Condition** | Any category failure rate > 10% |
| **Normal** | All categories < 5% |
| **Action** | Notify team channel, investigate category |
| **Possible causes** | Model weakness in specific area, new query pattern |
| **Runbook** | Review category samples, consider targeted improvements |

#### MED-02: Escalation Rate Anomaly

| Attribute | Value |
|-----------|-------|
| **Condition** | Tier 2→3 escalation rate > 15% or < 1% |
| **Normal** | ~5% |
| **Action** | Notify team, review autograder calibration |
| **Possible causes** | Autograder too aggressive/lenient, threshold misconfiguration |
| **Runbook** | Compare Tier 2 vs Tier 3 scores, adjust thresholds if needed |

#### MED-03: Autograder Bias Drift

| Attribute | Value |
|-----------|-------|
| **Condition** | Bias > +0.5 or < -0.5 |
| **Normal** | Between -0.2 and +0.2 |
| **Action** | Notify team, schedule autograder tuning |
| **Possible causes** | Prompt drift, rubric interpretation shift |
| **Runbook** | Review score distributions, compare to human labels |

---

### Low Alerts (Weekly Review)

These are tracked but don't require immediate action.

#### LOW-01: Throughput Below Expected

| Attribute | Value |
|-----------|-------|
| **Condition** | Tier 2 throughput 50-80% of expected |
| **Normal** | ~100% of expected |
| **Action** | Include in weekly report, monitor trend |

#### LOW-02: Tier 3 Overturn Rate Shift

| Attribute | Value |
|-----------|-------|
| **Condition** | Overturn rate outside 20-40% range |
| **Normal** | 20-40% |
| **Action** | Include in weekly report, review if persistent |

---

## Alert Routing

| Severity | Channel | Method |
|----------|---------|--------|
| Critical | On-call engineer | PagerDuty / phone call |
| High | On-call engineer | Slack DM + mention |
| Medium | #ai-quality channel | Slack message |
| Low | Weekly digest | Email report |

---

## Alert Suppression Rules

To avoid alert fatigue:

| Rule | Reason |
|------|--------|
| No duplicate alerts within 1 hour | Prevent spam during ongoing incident |
| Suppress during planned maintenance | Known downtime shouldn't page |
| Suppress Low alerts during High/Critical | Focus on the big problem first |
| Auto-resolve when metric recovers | Don't leave stale alerts open |

---

## Escalation Path

If an alert isn't acknowledged:

| Time | Action |
|------|--------|
| 0 min | Alert fires, primary on-call notified |
| 15 min | No ack → secondary on-call notified |
| 30 min | No ack → engineering manager notified |
| 1 hour | No ack → director notified |

---

## Alert Summary Table

| Alert ID | Condition | Severity | Response |
|----------|-----------|----------|----------|
| CRIT-01 | Safety > 0.5% | Critical | Page, investigate, possible rollback |
| CRIT-02 | Pipeline down 30 min | Critical | Page, restore pipeline |
| CRIT-03 | Quality crash during rollout | Critical | Auto-rollback, page |
| HIGH-01 | Quality < 4.0 (2 hrs) | High | Slack, investigate |
| HIGH-02 | Any dimension < 3.5 (2 hrs) | High | Slack, investigate dimension |
| HIGH-03 | Latency > 2 hrs | High | Slack, check pipeline |
| HIGH-04 | Agreement < 0.6 | High | Schedule review |
| MED-01 | Category > 10% failure | Medium | Team channel, review |
| MED-02 | Escalation rate anomaly | Medium | Team channel, calibrate |
| MED-03 | Bias > ±0.5 | Medium | Team channel, tune |
| LOW-01 | Throughput 50-80% | Low | Weekly report |
| LOW-02 | Overturn rate shift | Low | Weekly report |

---

## Post-Incident Review

After any Critical or High alert:

1. **Document** — What happened, when, impact
2. **Root cause** — Why did it happen?
3. **Fix** — What resolved it?
4. **Prevention** — How