# AI Evaluation Monitoring System

A design for monitoring AI assistant quality at production scale.

## The Problem

You have 10 million AI responses per day. How do you know if they're good? How do you catch quality regressions before users notice?

- Evaluating all 10M with humans: Impossible
- Evaluating all 10M with a large LLM: ~$100,000/day
- This design: ~$105/day with tiered evaluation

## What This Project Is

This is a **system design**, not running code. It documents how you would build an evaluation monitoring system for a production AI assistant.

It answers: **"Did response quality degrade in the last 24 hours, and if so, what's causing it?"**

## Documentation

| Document | What it covers |
|----------|----------------|
| [Architecture](docs/architecture.md) | Tiered evaluation pipeline, data flow, cost analysis |
| [Metrics](docs/metrics.md) | Quality, operational, and calibration metrics with thresholds |
| [CI/CD Pipelines](docs/ci-cd-pipelines.md) | Quality gates for model and autograder deployment |
| [Alerting](docs/alerting.md) | Alert definitions, severity levels, escalation paths |

## Key Concepts

### Tiered Evaluation

Not every response needs expensive evaluation. Use cheap checks for everything, expensive checks only when needed.

| Tier | What | Volume | Cost |
|------|------|--------|------|
| 1 | Rule-based checks | 10M/day (100%) | ~$0 |
| 2 | Fast LLM autograder | 100K/day (1% sample) | $30/day |
| 3 | Large LLM autograder | 5K/day (flagged) | $50/day |
| 4 | Human review | 50/day (edge cases) | $25/day |

### Quality Dimensions

Every response is scored on 5 dimensions (1-5 scale):

- **Correctness** — Is the information accurate?
- **Completeness** — Does it fully answer the question?
- **Conciseness** — Is it the right length?
- **Naturalness** — Does it sound helpful?
- **Safety** — Is it appropriate?

### CI/CD Quality Gates

Two pipelines with quality gates:

1. **Assistant Pipeline** — Golden set → Baseline comparison → Shadow deployment → Gradual rollout
2. **Evaluator Pipeline** — Human agreement check → Bias check → A/B comparison → Staged rollout

## Mapping to Traditional QE

If you come from traditional quality engineering:

| Traditional | AI Evaluation Equivalent |
|-------------|--------------------------|
| BVT | Golden set evaluation |
| Regression tests | Baseline comparison |
| Integration tests | Shadow deployment |
| Production monitoring | Tiered evaluation (Tiers 1-4) |

## Related Work

This design builds on concepts from:

- [My Autograder project](https://github.com/rasiulyte/assistant-autograder) — LLM-as-judge validation with prompt strategies
- [NIST AI RMF mapping](https://github.com/rasiulyte/nist-ai-rmf-evaluation) — Translating governance requirements to evaluation approaches

## Author

Rasa Rasiulytė  
Portfolio: [rasar.ai](https://rasar.ai)  
GitHub: [github.com/rasiulyte](https://github.com/rasiulyte)