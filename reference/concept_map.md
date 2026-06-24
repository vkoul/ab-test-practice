# A/B Testing Concept Map

## How Everything Connects

```
                        ┌─────────────────────────────────────┐
                        │       A/B TEST LIFECYCLE            │
                        └─────────────────────────────────────┘
                                        │
            ┌───────────────────────────┼───────────────────────────┐
            │                           │                           │
            ▼                           ▼                           ▼
    ┌───────────────┐          ┌───────────────┐          ┌───────────────┐
    │    DESIGN     │          │    EXECUTE    │          │   ANALYZE     │
    │   (Week 1)    │          │  (Week 2-3)   │          │  (Week 1-4)   │
    └───────────────┘          └───────────────┘          └───────────────┘
            │                           │                           │
    ┌───────┴───────┐          ┌───────┴───────┐          ┌───────┴───────┐
    │               │          │               │          │               │
    ▼               ▼          ▼               ▼          ▼               ▼
 Metrics        Power       Validity      Early         Results      Decision
 Selection    Analysis     Checks       Monitoring    Computation   Framework
    │               │          │               │          │               │
    │               │          │               │          │               │
    ▼               ▼          ▼               ▼          ▼               ▼
┌────────┐   ┌─────────┐ ┌─────────┐  ┌─────────┐ ┌─────────┐  ┌─────────┐
│Target  │   │Sample   │ │SRM/     │  │Peeking  │ │T-test   │  │Launch   │
│Guardrail│   │Size     │ │Balance  │  │Problem  │ │P-value  │  │No-Launch│
│Informative│   │Duration │ │Targeting│  │Novelty  │ │CI       │  │Iterate  │
└────────┘   └─────────┘ │SUTVA    │  │Effects  │ │Rel Lift │  └─────────┘
                          └─────────┘  └─────────┘ └─────────┘
```

---

## Topic Dependencies (What You Need Before What)

```
Week 1 Foundations
├── Metric Selection ─────────────────────────────────────┐
│   ├── Sensitivity (baseline rate)                       │
│   ├── Timeliness (measurement lag)                      │
│   └── Business Connection                               │
├── Power Analysis ───────────────────────────────────────┤
│   ├── Alpha (significance level)                        │
│   ├── Beta (power = 1 - beta)                           │
│   ├── MDE (minimum detectable effect)                   │
│   └── Sample Size Formula                               │ Everything feeds into
├── Test Duration = f(sample_size, traffic)                │ the DECISION
├── Statistical Testing                                   │
│   ├── T-test → p-value                                  │
│   └── Confidence Intervals                              │
└── SUTVA                                                 │
                                                          │
Week 2 Validity                                           │
├── Assignment Imbalance (SRM) ───────────────────────────┤
│   └── Chi-squared test                                  │
├── Targeting/Instrumentation ────────────────────────────┤
│   └── Subset to correct population                      │
├── Dimensional Breakdowns ───────────────────────────────┤
│   └── Segment by continent/device/history               │
└── Spillover Effects ────────────────────────────────────┤
                                                          │
Week 3 Early Results                                      │
├── Peeking Problem ──────────────────────────────────────┤
│   └── Multiple testing inflation                        │
├── Novelty Effects ──────────────────────────────────────┤
│   └── Time-based cohort analysis                        │
├── Underpowered Tests ───────────────────────────────────┤
│   └── Winner's curse                                    │
└── Stakeholder Management ───────────────────────────────┤
                                                          │
Week 4 Concurrent Tests                                   │
├── Multi-test Strategies ────────────────────────────────┤
│   ├── Sequential                                        │
│   ├── Mutual Exclusion                                  │
│   ├── Full Factorial                                    │
│   └── Overlapping                                       │
├── Interaction Effects ──────────────────────────────────┤
│   └── Check: effect_A_alone vs effect_A_with_B          │
└── Bonferroni Correction ────────────────────────────────┘
    └── Adjust alpha for multiple comparisons
```

---

## Decision Tree: "What Should I Do?"

```
START: You have A/B test results
│
├── Step 1: Is there assignment imbalance?
│   ├── YES → Investigate! Don't trust results until resolved.
│   └── NO ↓
│
├── Step 2: Was the test correctly targeted?
│   ├── NO → Subset to correct population, re-analyze
│   └── YES ↓
│
├── Step 3: Has the test run for the planned duration?
│   ├── NO → Is there a safety concern (guardrail tanking)?
│   │   ├── YES → Consider early stopping
│   │   └── NO → Keep running! Don't peek.
│   └── YES ↓
│
├── Step 4: Is the target metric stat-sig positive?
│   ├── NO → Don't launch. Consider: run longer / iterate / move on.
│   └── YES ↓
│
├── Step 5: Is any guardrail stat-sig NEGATIVE?
│   ├── YES → Don't launch. Investigate the guardrail hit.
│   └── NO ↓
│
├── Step 6: Are there concerning dimensional breakdowns?
│   ├── YES → Investigate. Maybe launch with conditions.
│   └── NO ↓
│
└── DECISION: LAUNCH ✓
```

---

## Common Pitfalls & What Catches You

| Pitfall | Week | What Goes Wrong | How to Catch It |
|---------|------|----------------|-----------------|
| Wrong metric choice | 1 | Optimize something that doesn't drive business value | Metric selection framework |
| Underpowered test | 1 | Run too short → false negatives or inflated effects | Always do power analysis first |
| Peeking at results | 3 | Stop early on a "significant" result that's actually noise | Pre-commit to end date |
| Ignoring SRM | 2 | Trust results from a broken randomization | Always check chi-squared first |
| Wrong targeting | 2 | Include users who never saw the treatment | Verify population matches design |
| Novelty effects | 3 | Ship a feature that only works because it's new | Time-based cohort analysis |
| Ignoring interactions | 4 | Ship A and B independently when they conflict together | Interaction check before shipping |
| Multiple comparisons | 4 | Find "significance" by testing many things | Bonferroni or pre-registration |

---

## The A/B Tester's Mindset

```
"Extraordinary claims require extraordinary evidence."

Before you launch, ask yourself:
1. Did anything go wrong with the test mechanics?     (Week 2)
2. Can I trust the effect size I'm seeing?            (Week 3)
3. Are there hidden interactions or subgroup effects? (Week 2, 4)
4. Can I explain these results to a skeptical colleague? (All weeks)
```
