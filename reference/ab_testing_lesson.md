# A/B Testing: The Complete Mental Model

## What is an A/B Test?

An A/B test is a **randomized controlled experiment** applied to product decisions. You split users into two groups:
- **Control** — sees the existing experience
- **Treatment** — sees the new change

Then you measure whether the change caused a meaningful difference in a metric you care about.

The key word is **caused**. Without randomization, you can't separate "the feature helped" from "the users who happened to use it were already different."

---

## The 5-Step Framework

Every A/B test follows this sequence:

```
1. DEFINE   → What metric? What's the hypothesis?
2. SIZE     → How many users? How long?
3. RUN      → Randomize, wait, don't peek
4. VALIDATE → Did anything go wrong?
5. DECIDE   → Launch, iterate, or kill?
```

Let's walk through each.

---

## Step 1: DEFINE — Metrics & Hypothesis

### Picking Your Target Metric

You need a metric that balances three things:

| Criterion | Question to Ask |
|-----------|----------------|
| **Sensitivity** | Can I detect a change with reasonable sample size? (Higher baseline = easier) |
| **Timeliness** | How long until I can measure it? (Days vs. weeks) |
| **Business value** | If I improve this, does the company actually benefit? |

**Example:** You're improving an onboarding flow. Options:
- Sign-up completion (40% baseline, 2hr lag) — sensitive but low value
- First purchase (20% baseline, 7-day lag) — balanced
- Retention at 30 days (10% baseline, 30-day lag) — valuable but slow

Usually the middle option wins.

### Guardrail Metrics

These are metrics that must NOT get worse. They protect you from unintended harm.

Example: If you make sign-up easier (target: completions ↑), your guardrail might be cancellation rate — because if you're attracting low-quality users, cancellations will spike.

### The Hypothesis

Write it as: **"If [change], then [metric] will [direction] because [mechanism]."**

The "because" is critical — it forces you to articulate WHY you think this will work.

---

## Step 2: SIZE — Power Analysis

The most common beginner mistake: **running a test too short**.

You need enough users to distinguish a real effect from random noise. This is called **statistical power**.

### The Key Formula

For a binary metric (yes/no, like "did they convert?"):

```
n_per_group = (Z_α/2 + Z_β)² × [p₁(1-p₁) + p₂(1-p₂)] / (p₂ - p₁)²
```

In plain English, sample size depends on:
- **Baseline rate** (p₁) — higher = need fewer users
- **Effect size** you want to detect — smaller effects need MORE users
- **Confidence level** (α = 0.05 typically) — higher confidence = more users
- **Power** (1-β = 0.80 typically) — higher power = more users

### Practical Example

```python
import scipy.stats as stats
import math

baseline = 0.20          # 20% conversion rate
mde_relative = 0.05     # want to detect 5% relative lift
p1 = baseline            # 0.20
p2 = baseline * (1.05)   # 0.21

z_alpha = 1.96           # for α = 0.05, two-sided
z_beta = 0.84            # for power = 0.80

n = ((z_alpha + z_beta)**2 * (p1*(1-p1) + p2*(1-p2))) / (p2 - p1)**2
print(f"Need {math.ceil(n):,} users per group")  # ~61,600
```

### Duration

```
Days needed = (n_per_group × 2) / daily_traffic
```

If you get 5,000 users/day: 123,200 / 5,000 = **25 days → round to 28 (4 full weeks)**

Always round to full weeks to avoid day-of-week bias.

---

## Step 3: RUN — The Hard Part is Patience

Once the test is live:

**DO:**
- Monitor for bugs/crashes (safety)
- Check that randomization is working (equal split)

**DON'T:**
- Look at the metric results before the planned end date
- Stop early because "it looks significant"

### Why Not Peek?

If you check results 5 times during a test, your false positive rate jumps from 5% to ~14%. The math of repeated testing means random fluctuations WILL cross the significance threshold temporarily. If you stop at that moment, you'll ship a change that doesn't actually work.

---

## Step 4: VALIDATE — Did Anything Go Wrong?

Before trusting results, run this checklist:

### 4a. Assignment Imbalance (Sample Ratio Mismatch)

Did users split evenly? Use a chi-squared test:
```python
from scipy.stats import chisquare
observed = [n_control, n_treatment]
expected = [total/2, total/2]
_, p_value = chisquare(observed, f_exp=expected)
# p < 0.05 → something is broken. Investigate.
```

### 4b. Correct Targeting

Were only the intended users included? If your test targets mobile users but desktop users leaked in, the data is diluted.

### 4c. Novelty Effects

Did the effect decay over time? Plot lift by cohort day:
- **Declining** → users were just curious about the change, not genuinely helped
- **Stable** → real effect, safe to trust

---

## Step 5: DECIDE — The Launch Framework

After the test completes and validation passes:

```
IF   target metric is stat-sig positive (p < 0.05)
AND  guardrail is NOT stat-sig negative
THEN → LAUNCH

IF   target is not significant
THEN → DON'T LAUNCH (iterate or move on)

IF   guardrail is stat-sig negative
THEN → DON'T LAUNCH (regardless of target)
```

### Reading the Results

```python
# You'll see output like:
# rel_diff=0.05, abs_diff=0.01, p=0.02, CI=[1.2%, 8.8%]

# This means:
# - Treatment is 5% relatively better than control
# - p=0.02 → statistically significant (< 0.05)
# - CI doesn't cross zero → confirms significance
# - We're 95% confident the true lift is between 1.2% and 8.8%
```

---

## Common Traps (What Separates Beginners from Experts)

| Trap | What Happens | The Fix |
|------|-------------|---------|
| Running underpowered | You miss real effects, or overestimate the ones you find | Always do power analysis FIRST |
| Peeking | You ship things that don't work | Commit to end date upfront |
| Ignoring guardrails | You improve one metric while breaking something else | Always track guardrails |
| Shipping on early data | Novelty effects inflate early results | Wait for full duration |
| p=0.06 means "no effect" | It means "insufficient evidence" — different thing | Look at CI width, consider running longer |
| Multiple segments | Testing 10 segments at α=0.05 gives ~40% chance of a false positive somewhere | Use for exploration, not confirmation |

---

## Where to Go Next

You have the full course materials ready. Suggested order:

1. **Read** `study_notes/week01_study_designing_ab_tests.ipynb`
2. **Attempt** `assignments/week01_assignment_designing_ab_tests.ipynb`
3. **Check** against `solutions/w01_designing_ab_tests.ipynb`
4. Repeat for weeks 2-4

Each week adds complexity: Week 2 (validity threats), Week 3 (early results & stakeholder pushback), Week 4 (concurrent tests & interactions).
