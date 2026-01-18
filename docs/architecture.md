# Architecture: AI Evaluation Monitoring System

## Overview

A monitoring system that answers: **"Did response quality degrade in the last 24 hours, and if so, what's causing it?"**

Designed for a conversational AI assistant at scale:
- 10M daily responses
- ~$105/day evaluation cost
- Quality scores within 1 hour of production responses

---

## System Context
```
┌─────────────────────┐
│  Conversational AI  │  ← Not part of this system
│  (e.g., assistant)  │
└──────────┬──────────┘
           ↓
     Response Logs
           ↓
┌─────────────────────┐
│  Evaluation         │  ← THIS PROJECT
│  Monitoring System  │
└──────────┬──────────┘
           ↓
    Quality Metrics
           ↓
┌─────────────────────┐
│  Dashboards/Alerts  │
│  CI/CD Quality Gates│
└─────────────────────┘
```

This system **observes** the AI assistant's outputs. It does not generate responses—it evaluates them.

---

## The Problem: Why Tiers?

You have 10 million AI responses per day. You need to know if they're good or bad.

- **Evaluating all 10M with humans:** Impossible—would cost millions
- **Evaluating all 10M with a large AI:** ~$100,000/day—too expensive
- **Solution:** Use cheap/fast checks for everything, expensive checks only when needed

This is called **tiered evaluation**.

---

## Tiered Evaluation Pipeline

Think of it like hospital triage:

| Hospital | Evaluation System |
|----------|-------------------|
| Front desk checks insurance, takes temperature | Tier 1 runs fast checks on all responses |
| Nurse handles routine cases | Tier 2 uses cheap AI on a sample |
| Doctor sees concerning cases | Tier 3 uses expensive AI on flagged responses |
| Specialist handles complex cases | Tier 4 uses humans for edge cases |

Each tier is a filter. Most responses pass Tier 1 and never need more.

---

### Tier 1: Rule-Based Checks (100% of traffic)

Fast, cheap filters that catch obvious problems.

| Check | What it catches | Cost |
|-------|-----------------|------|
| Response length | Too short (<10 chars) or too long (>5000 chars) | ~$0 |
| Blocked patterns | Known harmful phrases, PII patterns | ~$0 |
| Format validation | Empty responses, malformed output | ~$0 |
| Latency flags | Responses that took >10s | ~$0 |

**Output:** Logs metadata for all 10M responses. Passes 1% random sample to Tier 2.

---

### Tier 2: Fast LLM Autograder (1% sample = 100K/day)

A small, fast, cheap AI model scores responses on 5 dimensions.

| Dimension | What it measures |
|-----------|------------------|
| Correctness | Is the information accurate? |
| Completeness | Does it fully answer the question? |
| Conciseness | Is it the right length for the query? |
| Naturalness | Does it sound like a helpful assistant? |
| Safety | Is it appropriate and not harmful? |

**Model:** Claude Haiku or GPT-4o-mini  
**Cost:** ~$0.0003/eval × 100K = ~$30/day  
**Output:** Scores 1-5 per dimension. Flags responses with any score <3.

---

### Tier 3: Large LLM Autograder (5% of Tier 2 flags = ~5K/day)

A larger, smarter, more expensive AI model re-evaluates flagged responses.

**Why?** Cheap AI makes mistakes. Expensive AI double-checks before escalating to humans.

**Model:** Claude Sonnet or GPT-4o  
**Cost:** ~$0.01/eval × 5K = ~$50/day  
**Output:** Confirmed failures go to Tier 4. Overturned flags logged for calibration.

---

### Tier 4: Human Review (~1% of Tier 3 = ~50/day)

Humans handle the hardest cases and keep the autograder accurate.

**Purpose:**
1. Catch failures the autograder misses
2. Generate ground truth labels to recalibrate Tiers 2 and 3

**Cost:** ~$0.50/eval × 50 = ~$25/day

---

## Data Flow Diagram
```
Production AI Assistant
         ↓
   Response Logs (10M/day)
         ↓
┌────────────────────────────────┐
│ Tier 1: Rule-based (100%)      │ ──→ Log metadata
└───────────┬────────────────────┘
            ↓ 1% sample
┌────────────────────────────────┐
│ Tier 2: Fast LLM (100K/day)    │ ──→ Quality scores
└───────────┬────────────────────┘
            ↓ Flagged (<3 on any dimension)
┌────────────────────────────────┐
│ Tier 3: Large LLM (5K/day)     │ ──→ Confirmed/overturned
└───────────┬────────────────────┘
            ↓ High-confidence failures
┌────────────────────────────────┐
│ Tier 4: Human Review (50/day)  │ ──→ Ground truth labels
└───────────┬────────────────────┘
            ↓
┌────────────────────────────────┐
│ Metrics Database               │
└───────────┬────────────────────┘
            ↓
   Dashboards ← Alerts ← CI/CD Gates
```

---

## Cost Summary

| Tier | Coverage | Daily Volume | Cost/Eval | Daily Cost |
|------|----------|--------------|-----------|------------|
| 1 | 100% | 10M | ~$0 | ~$0 |
| 2 | 1% sample | 100K | $0.0003 | $30 |
| 3 | 5% of flags | 5K | $0.01 | $50 |
| 4 | 1% of Tier 3 | 50 | $0.50 | $25 |
| **Total** | | | | **~$105/day** |

**Compared to evaluating all 10M with a large model (~$100K/day), this is a 99.9% cost reduction.**

---

## Mapping to Traditional Testing

If you come from a traditional QE background, here's how this maps:

| Traditional Test Type | AI Evaluation Equivalent | When it runs |
|-----------------------|--------------------------|--------------|
| **BVT** | Golden set evaluation | Before deployment |
| **Regression** | Baseline comparison | Before deployment |
| **Integration** | Shadow deployment | During staged rollout |
| **Production Monitoring** | Tiered evaluation (Tiers 1-4) | Continuously in prod |

### Tier 1 as Traditional Tests

Tier 1 rule-based checks map directly to familiar test types:

| Test Type | Tier 1 Check | Example |
|-----------|--------------|---------|
| BVT | Critical format checks | Response exists, not empty, within length limits |
| Regression | Known-bad pattern detection | Blocked phrases from past incidents |
| Integration | System health checks | Latency < 10s, no timeout errors |
| Smoke | Basic sanity checks | Valid encoding, no malformed output |

### Key Difference from Traditional Testing

| Traditional | AI Evaluation |
|-------------|---------------|
| Deterministic (same input → same output) | Probabilistic (outputs vary) |
| Pass/fail is binary | Quality is a score (1-5) |
| Test cases written once | Continuously discover new failures |
| Bugs fixed in code | Failures may need retraining or prompt changes |

---

## Key Design Decisions

### Why sample 1% for Tier 2?

At 10M daily responses, 1% gives 100K samples—statistically sufficient to detect a 0.1% change in failure rate with 95% confidence.

### Why use LLM autograders instead of classifiers?

Fine-tuned classifiers are faster and cheaper but need labeled training data for each failure type. LLM autograders generalize to new failure modes without retraining.

### Why keep humans in the loop?

LLM autograders drift over time and miss edge cases. Human review catches what automation misses and generates labels to recalibrate the system.

---

## Next Steps

- [Metrics definitions](metrics.md) — what we measure and alert on
- [CI/CD pipelines](ci-cd-pipelines.md) — quality gates for deployment
- [Alerting logic](alerting.md) — when and how we notify