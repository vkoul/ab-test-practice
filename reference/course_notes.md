# A/B Testing in Practice — Comprehensive Course Notes

---

## Week 1: Designing A/B Tests

### 1.1 The FamilyNest Context
- Company: Airbnb-like platform for families with young kids
- Role: Product Data Scientist on host growth team
- Goal: Improve onboarding flow to grow the number of homes on platform
- PM: PaM (later replaced by Max in Week 3)

### 1.2 The Host Onboarding Funnel

| Step | Conv. from Last | Conv. from Start | Median Time |
|------|-----------------|------------------|-------------|
| Start onboarding | — | — | — |
| Publish home (listing active) | 40% | 40% | 2 hours |
| Home booked by guest | 50% | 20% | 7 days |
| First guest stays | 90% | 18% | 14 days |
| First review received | 60% | 10.8% | 2 days |

### 1.3 Metric Selection

**Three types of metrics in an A/B test:**
1. **Target metric** — the one you optimize for; determines launch/no-launch
2. **Guardrail metric** — must NOT degrade; protects against harm
3. **Informative metric** — useful for learning, not decision-making

**Criteria for choosing a target metric:**

| Criterion | What It Means | Trade-off |
|-----------|--------------|-----------|
| Sensitivity | Higher baseline rate → easier to detect small changes | Metrics closest to the change are most sensitive but least valuable |
| Timeliness | Shorter lag to measure → faster decisions | Downstream metrics take longer but matter more |
| Business connection | Clear link to revenue/growth | The most valuable metrics are often hardest to move |

**For FamilyNest onboarding:**
- **New Active Listings (NAL):** Most sensitive, most timely (2hr), least valuable
- **New Booked Listings (NBL):** Balanced — moderate sensitivity, 7-day lag, clear revenue connection
- **New Reviewed Listings (NRL):** Most valuable, but 23-day lag and lowest sensitivity

**Best choice: NBL** — balances the triad. NAL is acceptable if other teams already own NBL.

**Guardrail examples:**
- If target = NBL → guardrail = New Cancelled Listings (NCL)
- If target = NAL → guardrail = NBL (ensure quality isn't dropping)

### 1.4 Hypothesis Formulation

**Format:** "If [specific change], then [metric] will [increase/decrease] by [amount] because [mechanism]"

**Example:** "If we auto-generate listing titles from address + room count, then NBL will increase by 5% because reducing friction in the title step will increase completion rates."

### 1.5 Sample Size Calculation

**Key parameters:**
- α (alpha) = significance level, typically 0.05
- β (beta) = false negative rate; Power = 1 - β, typically 0.80
- p₁ = baseline conversion rate
- MDE = minimum detectable effect (relative or absolute)
- p₂ = p₁ × (1 + relative_MDE)

**Formula (binary metrics):**
```
n_per_variant = (Z_{α/2} + Z_β)² × [p₁(1-p₁) + p₂(1-p₂)] / (p₂ - p₁)²
```

Where Z_{α/2} = 1.96, Z_β = 0.84 (for 80% power)

**Quick approximation:** n ≈ 16 × p(1-p) / δ² where δ is the absolute MDE

### 1.6 Test Duration

```
Duration (days) = (n_per_variant × number_of_variants) / daily_eligible_traffic
```

**Always round UP to full weeks** to avoid day-of-week effects.

**Example:** NBL baseline = 20%, MDE = 5% relative (= 1% absolute), 5000 users/day
- n ≈ 61,000 per variant → total = 122,000
- Duration = 122,000 / 5,000 = 24.4 days → round to 28 days (4 weeks)

### 1.7 Analyzing Results

**Core calculations:**
- Relative lift = (mean_treatment - mean_control) / mean_control
- Absolute lift = mean_treatment - mean_control
- P-value from two-sided t-test (scipy.stats.ttest_ind)
- 95% CI on the absolute difference, then divide by mean_control for relative CI

**Interpretation:**
- p < 0.05 → statistically significant (reject null hypothesis)
- p ≥ 0.05 → NOT significant (cannot reject null; does NOT mean "no effect")
- CI not crossing zero → significant

### 1.8 Launch Decision Framework

| Target | Guardrail | Decision |
|--------|-----------|----------|
| Stat-sig positive | Not sig negative | **LAUNCH** |
| Stat-sig positive | Stat-sig negative | **DON'T LAUNCH** (investigate) |
| Not significant | Any | **DON'T LAUNCH** (iterate or run longer) |
| Stat-sig negative | Any | **DON'T LAUNCH** |

### 1.9 SUTVA (Stable Unit Treatment Value Assumption)

**Definition:** The treatment assigned to one unit does not affect the outcome of any other unit.

**Violations in marketplaces:**
- If treatment makes listing easier → more supply → lower prices → affects control group bookings
- Network effects: one user's positive experience brings referrals into control/treatment
- Shared resources: if control and treatment compete for the same guests

**Mitigation:** Cluster randomization (randomize by geography or time), or accept the bias and note it.

---

## Week 2: Handling Test Validity Threats

### 2.1 Types of Validity

- **Internal validity:** Can we trust that the treatment CAUSED the observed difference?
- **External validity:** Can we generalize the result beyond this specific test?

### 2.2 Assignment Imbalance (Sample Ratio Mismatch / SRM)

**What:** The actual ratio of users in control vs. treatment doesn't match the intended split.

**Why it matters:** Indicates a systematic bias in randomization (a bug). If assignment is biased, so are the results.

**Detection:** Chi-squared goodness-of-fit test
```python
from scipy.stats import chisquare
observed = [n_control, n_treatment]
expected = [total/2, total/2]
chi2, p_value = chisquare(observed, f_exp=expected)
```

**Decision rule:**
- p > 0.05 → no evidence of imbalance, proceed
- p < 0.05 → investigate before trusting results

**Also check by segment:** Run chi-squared within each continent, device type, etc. An overall balance can mask segment-level imbalance.

### 2.3 Instrumentation / Targeting Errors

**Problem:** Users who were never supposed to see the treatment end up in the test data.

**Classic example (Africa map test):**
- New map service only affects African addresses
- But the test data includes users from ALL continents
- Untargeted users dilute the treatment effect

**Detection:**
- Check which segments are in the data vs. which were targeted
- If users outside the target population are included → subset to correct population

**Resolution:** Filter to correctly-targeted users, then re-analyze. The valid results come from the intended population only.

### 2.4 Dimensional Breakdowns

**Purpose:** Detect heterogeneous treatment effects (HTE) — the treatment might help one group but hurt another.

**Common dimensions to check:**
- Continent/region
- Device (web/iOS/Android)
- User history (booked_previously: 0 or 1)
- Time of sign-up

**Interpretation:**
- If effect is consistent across segments → strong evidence of a real effect
- If effect only exists in one segment → investigate (could be real targeting opportunity or a bug)

**Caveat:** Multiple comparisons. Testing many segments inflates false positive rate. Use these as exploratory, not confirmatory.

### 2.5 Spillover / Interference Effects

**When one user's treatment affects another user's outcome:**
- Supply side: more listings from treatment → more competition for control listings
- Demand side: better listings from treatment → steal guests from control hosts
- Platform effects: platform reputation improves → affects all users

**Mitigation strategies:**
- Cluster-based randomization (by geo/market)
- Time-based experiments (on/off by week)
- Accept small bias and acknowledge it

### 2.6 Complete Analysis Workflow (Checklist)

```
□ 1. Check assignment imbalance (overall + by segment)
□ 2. Verify correct targeting/triggering
□ 3. Analyze TARGET metric (rel_diff, abs_diff, p-value, CI)
□ 4. Analyze GUARDRAIL metric(s)
□ 5. Break down by key dimensions
□ 6. Make recommendation with reasoning
```

---

## Week 3: Safely Handling Early Results

### 3.1 The Peeking Problem

**The temptation:** "Results look great at day 5, let's just ship!"

**The reality:** If you peek N times during a test, your effective alpha is MUCH higher than 0.05.
- Peek once → α ≈ 0.08
- Peek 5 times → α ≈ 0.14
- Continuous monitoring → α approaches 1.0 over time

**Why:** At any point during a test, random fluctuation can make the p-value dip below 0.05 temporarily, even when there's no real effect. If you stop when it dips, you'll have a high false positive rate.

**Analogy:** Flipping a fair coin — if you stop counting whenever you're ahead, you'll conclude the coin is biased.

### 3.2 When Early Stopping IS Okay

1. **Safety:** Guardrail metric is clearly tanking → stop to protect users
2. **Pre-specified sequential testing:** Alpha-spending functions (O'Brien-Fleming, Pocock) that account for multiple looks
3. **Obvious harm:** Bugs, crashes, clearly broken experiences
4. **Futility:** Test is so underpowered given observed effect that continuing is pointless

### 3.3 Novelty Effects

**Definition:** Users interact more with a new feature simply because it's NEW, not because it's better.

**Characteristics:**
- Metric lift is highest in the first few hours/days
- Lift decays over time as users habituate
- If you measure only early data, you'll overestimate the true long-run effect

**Detection method:**
1. Group users by how long they've been in the experiment (cohort hour/day)
2. Plot relative lift vs. time-in-experiment
3. **Declining trend** → novelty effect (early results inflated)
4. **Stable trend** → genuine effect (safe to trust)

**The opposite — Primacy effect:** Users initially resist change but adapt over time. Lift INCREASES over time. Less common but possible.

### 3.4 Winner's Curse / Underpowered Early Looks

**Problem:** If you stop an underpowered test early because it looks significant:
- The observed effect is systematically inflated (winner's curse)
- You may ship something with a much smaller real effect than measured
- Or ship something with NO real effect (Type I error)

**Why:** Among all possible outcomes at an underpowered sample size, only the ones with inflated effects cross the significance threshold. You're selecting for overestimates.

**Rule:** You can only trust effects that are at or above your pre-computed MDE. Effects well above MDE from a small sample deserve skepticism, not celebration.

### 3.5 Stakeholder Management

**Common scenario:** PM says "The numbers look great at day 3! Let's ship!"

**How to push back professionally:**

1. **Acknowledge** the enthusiasm: "I'm excited too — early signals are promising."
2. **Explain** the risk simply: "Shipping now risks shipping something that actually hurts us in the long run."
3. **Provide evidence:** Show novelty effect analysis or power analysis
4. **Recommend** a concrete timeline: "Let's check again at day X when we'll have enough data to be confident."

**Key arguments:**
- Novelty effects inflate early metrics (show the time plot)
- Underpowered tests → unreliable effect sizes
- Day-of-week effects need full weekly cycles to stabilize
- Regression to the mean is real

### 3.6 Handling Inconclusive Results

**p > 0.05 does NOT mean "no effect"** — it means "insufficient evidence."

**Options:**
1. Run longer for more power (if practical)
2. Accept the null and move on (if opportunity cost is high)
3. Iterate on the design and re-test (if you have a hypothesis for why it didn't work)
4. Reduce MDE and accept you need a bigger sample

**How to communicate:** "The test did not reach statistical significance. We cannot confidently say the feature improves [metric]. Our confidence interval includes both positive and negative outcomes."

### 3.7 The Decision Matrix

| Scenario | Action |
|----------|--------|
| Early results positive, test still running | Keep running |
| Early results negative on guardrail | Consider stopping for safety |
| Stakeholder wants to ship early | Push back with evidence |
| Full duration, barely significant | Launch cautiously, monitor |
| Full duration, not significant | Don't launch, iterate |
| Full duration, stat-sig + no guardrail hit | Launch |

---

## Week 4: Running Concurrent Tests

### 4.1 The Challenge

Multiple ideas in the pipeline → pressure to test them simultaneously. When two changes target the **same surface** (same page/flow), they may interact.

**Key distinction:**
- Different surfaces (e.g., landing page + checkout page) → safe to overlap
- Same surface (e.g., two changes to onboarding page) → need careful handling

### 4.2 Four Strategies for Same-Surface Tests

#### Option A: Sequential Testing
- Run Test 1 → analyze → then run Test 2
- **Pro:** Clean, no interaction concerns
- **Con:** 2× the time; Test 2's baseline changes if Test 1 ships

#### Option B: Mutual Exclusion (Traffic Splitting)
- Split traffic: 50% eligible for Test A, 50% for Test B
- Each test has its own internal 50/50 split
- **Pro:** Simultaneous, no interactions
- **Con:** Half the traffic per test → longer duration or larger MDE

#### Option C: Full Factorial (Multi-arm)
- All combinations tested: Control, A-only, B-only, A+B
- **Pro:** Can detect interactions; most informative
- **Con:** Needs more traffic (4 cells); complex analysis

#### Option D: Overlapping (with interaction check)
- Both tests run on 100% of traffic independently
- Users can be in both tests simultaneously
- **Pro:** Full power for each test
- **Con:** If interactions exist, results are biased; must check after

### 4.3 Sample Size for Multi-arm Tests

With K variants:
- Need Bonferroni correction: α_adjusted = α / number_of_comparisons
- For 3 variants (2 comparisons to control): use α = 0.025 per comparison
- Traffic per variant = daily_traffic / K
- Duration increases proportionally

**Example:** 3 variants, 4000 users/day → 1333/variant/day. If n=61,000 per variant:
- Duration = 61,000 / 1,333 = 46 days → 7 weeks

### 4.4 Interaction Effects

**Definition:** The effect of change A depends on whether change B is also present.

**Example:** Price algorithm (A) + Discount checkbox (B)
- A alone increases bookings by 3%
- B alone increases bookings by 2%
- A+B together increases bookings by only 3% (not 5%)
- → Negative interaction: the effects don't add up

**How to detect (in overlapping tests):**
```
1. Merge datasets on user_id
2. Create 4 groups: (ctrl_A, ctrl_B), (treat_A, ctrl_B), (ctrl_A, treat_B), (treat_A, treat_B)
3. Effect of A alone = mean(treat_A, ctrl_B) - mean(ctrl_A, ctrl_B)
4. Effect of A given B = mean(treat_A, treat_B) - mean(ctrl_A, treat_B)
5. If these differ significantly → interaction exists
```

### 4.5 Decisions with Interactions

| Finding | Recommendation |
|---------|---------------|
| No interaction, both positive | Ship both |
| No interaction, one positive one flat | Ship the positive one |
| Positive interaction (effects amplify) | Ship both together |
| Negative interaction | Pick the better one, OR find a way to resolve the conflict |
| One helps, one hurts | Ship the helpful one, kill the other |

### 4.6 Adapted Analysis for Multi-arm

The `calculate_results()` function changes to use `!= "control"` instead of `== "treatment"`:
```python
mean_treatment = df.loc[df['variant'] != "control", metric].mean()
```

For specific comparisons in a multi-arm test, filter to the two groups being compared:
```python
df_subset = df[df['variant'].isin(['control', 'treatment_1'])]
```

---

## Cross-Cutting Themes

### Communication Principles
1. Always lead with the recommendation, then supporting evidence
2. Quantify: "5% relative lift" not "it improved"
3. Acknowledge uncertainty: include confidence intervals
4. Anticipate questions: cover guardrails and edge cases proactively
5. Make it actionable: "I recommend we [action] because [reason]"

### Common Mistakes to Avoid
1. Choosing a metric that's too far from the product change (low sensitivity)
2. Not running a power analysis before starting the test
3. Peeking at results and stopping early
4. Ignoring assignment imbalance checks
5. Not verifying that targeting worked correctly
6. Treating p > 0.05 as "the feature doesn't work"
7. Shipping based on underpowered early results with inflated effects
8. Running concurrent tests on the same surface without checking interactions
9. Not doing dimensional breakdowns to catch segment-specific issues
10. Failing to push back on stakeholders who want to ship prematurely

### The A/B Tester's Hierarchy of Evidence
```
Strongest: Full-duration, adequately powered, no validity threats, significant result
     ↓
Strong: Full-duration, significant, one minor validity concern (documented)
     ↓
Moderate: Adequate power but result barely significant (CI close to zero)
     ↓
Weak: Underpowered, or early results only, or validity threats present
     ↓
Weakest: Peeked early, no power analysis, multiple validity issues
```
