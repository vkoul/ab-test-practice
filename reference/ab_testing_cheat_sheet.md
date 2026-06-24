# A/B Testing Cheat Sheet

## 1. Key Terminology

| Term | Definition |
|------|-----------|
| **Control** | The existing experience (baseline) |
| **Treatment** | The new experience being tested |
| **Target Metric** | Primary metric used to decide launch/no-launch |
| **Guardrail Metric** | Metric that must NOT degrade significantly |
| **MDE** | Minimum Detectable Effect - smallest lift worth detecting |
| **Statistical Significance** | p-value < alpha (typically 0.05) |
| **Power** | Probability of detecting a true effect (typically 80%) |
| **Alpha (α)** | False positive rate (typically 5%) |
| **Beta (β)** | False negative rate (typically 20%) |
| **SUTVA** | Stable Unit Treatment Value Assumption |
| **SRM** | Sample Ratio Mismatch (assignment imbalance) |

---

## 2. Metric Selection Framework

### Target Metric Criteria
1. **Sensitivity**: Higher baseline rate → easier to detect changes
2. **Timeliness**: Shorter measurement lag → faster iteration
3. **Business Connection**: Clear link to revenue/growth

### Guardrail Metric Purpose
- Protect against unintended negative consequences
- Example: If target = New Booked Listings, guardrail = New Cancelled Listings

---

## 3. Sample Size Formula (Binary Metrics)

```
n = (Z_{α/2} + Z_β)² × [p₁(1-p₁) + p₂(1-p₂)] / (p₂ - p₁)²
```

Where:
- `Z_{α/2}` = 1.96 (for α = 0.05, two-sided)
- `Z_β` = 0.84 (for power = 0.80)
- `p₁` = baseline conversion rate
- `p₂` = baseline × (1 + relative MDE)

### Quick Python Calculation
```python
import scipy.stats as stats
import math

def sample_size(baseline, mde_relative, alpha=0.05, power=0.80):
    p1 = baseline
    p2 = baseline * (1 + mde_relative)
    z_alpha = stats.norm.ppf(1 - alpha/2)
    z_beta = stats.norm.ppf(power)
    n = ((z_alpha + z_beta)**2 * (p1*(1-p1) + p2*(1-p2))) / (p2-p1)**2
    return math.ceil(n)

# Example: baseline=0.20, detect 5% relative lift
n = sample_size(0.20, 0.05)  # ~62,000 per variant
```

### Test Duration
```
Duration (days) = (sample_size_per_variant × num_variants) / daily_traffic
```
Always round UP to full weeks to avoid day-of-week bias.

---

## 4. Analysis Workflow (Checklist)

```
□ 1. Check assignment imbalance (chi-squared test, p > 0.05 = OK)
□ 2. Verify correct targeting/triggering
□ 3. Analyze TARGET metric
     → rel_diff, abs_diff, p-value, 95% CI
□ 4. Check GUARDRAIL metrics
□ 5. Break down by dimensions (if available)
□ 6. Check for novelty effects (if early results)
□ 7. Make launch/no-launch recommendation
```

---

## 5. Decision Rules

| Target Result | Guardrail Result | Decision |
|--------------|-----------------|----------|
| Stat-sig positive | Not sig negative | **LAUNCH** ✓ |
| Stat-sig positive | Stat-sig negative | **DON'T LAUNCH** ✗ (investigate) |
| Not significant | Any | **DON'T LAUNCH** ✗ (iterate) |
| Stat-sig negative | Any | **DON'T LAUNCH** ✗ |

---

## 6. Key Statistical Tests

### Two-Sample T-Test (for A/B test results)
```python
from scipy import stats
t_stat, p_value = stats.ttest_ind(control_data, treatment_data, equal_var=True)
```

### Chi-Squared Test (for assignment balance)
```python
from scipy.stats import chisquare
observed = df['variant'].value_counts().values
expected = [len(df)/2] * 2
chi2, p_value = chisquare(observed, f_exp=expected)
# p < 0.05 → imbalance detected (investigate!)
```

### Confidence Interval (relative lift)
```python
def get_ci(mean_t, mean_c, n_t, n_c, ci=0.95):
    sd = ((mean_t*(1-mean_t))/n_t + (mean_c*(1-mean_c))/n_c)**0.5
    lift = mean_t - mean_c
    val = stats.norm.isf((1-ci)/2)
    return (lift - val*sd, lift + val*sd)
```

---

## 7. Validity Threats & How to Detect

| Threat | Detection Method | Action |
|--------|-----------------|--------|
| **Assignment Imbalance (SRM)** | Chi-squared test on variant counts | Investigate randomization bug |
| **Incorrect Targeting** | Check dimensions; users outside target in data? | Subset to correct population |
| **Novelty Effect** | Plot lift over time (by cohort hour/day) | Wait for full duration |
| **Spillover / SUTVA violation** | Domain knowledge; check network effects | Consider cluster randomization |
| **Multiple Comparisons** | Bonferroni correction (α/k) | Adjust significance threshold |

---

## 8. Novelty Effect Detection

```python
# If lift DECREASES over time → novelty effect
# If lift STABILIZES → genuine effect
def check_novelty(df, metric, time_col='hrs_to_cohort'):
    results = []
    for t in range(1, int(df[time_col].max()) + 1):
        subset = df[df[time_col] <= t]
        t_mean = subset[subset['variant']=='treatment'][metric].mean()
        c_mean = subset[subset['variant']=='control'][metric].mean()
        results.append(((t_mean - c_mean) / c_mean) * 100)
    # Plot results - declining trend = novelty
```

---

## 9. Concurrent Tests

| Approach | When to Use | Traffic Impact |
|----------|-------------|---------------|
| **Sequential** | Low traffic; changes might interact | No impact (but 2× time) |
| **Mutual Exclusion** | Same surface; moderate traffic | Each test gets ½ traffic |
| **Full Factorial** | Same surface; high traffic; want interaction data | Need traffic for all cells |
| **Overlapping** | Different surfaces; or same with interaction check | Full traffic for each |

### Checking Interactions (Overlapping Tests)
```python
# Effect of A when B is absent
effect_a_alone = treatment_a_only.mean() - control.mean()
# Effect of A when B is present  
effect_a_with_b = both_treatments.mean() - treatment_b_only.mean()
# If these differ significantly → interaction exists
```

---

## 10. Stakeholder Communication Templates

### Test is NOT significant:
> "The test did not reach statistical significance (p = X.XX). We cannot confidently say the feature improves [metric]. Options: (1) Run longer for more power, (2) Iterate on the design, (3) Move to other priorities."

### Early results look great but test isn't done:
> "While early results are promising, shipping now risks: (1) Novelty effects inflating the metric, (2) Underpowered results being unreliable. I recommend running to the planned end date of [date]. Here's the novelty analysis showing [stable/declining] effects."

### Guardrail is hit:
> "The target metric shows a [X]% improvement, BUT the guardrail metric [name] shows a statistically significant [Y]% degradation. I recommend NOT launching until we understand and address the guardrail impact."

---

## 11. Common Formulas Quick Reference

| What | Formula |
|------|---------|
| Relative Lift | `(mean_treatment - mean_control) / mean_control` |
| Absolute Lift | `mean_treatment - mean_control` |
| Standard Error (binary) | `√(p₁(1-p₁)/n₁ + p₂(1-p₂)/n₂)` |
| Test Duration | `(n_per_variant × k_variants) / daily_traffic` |
| Bonferroni α | `α / number_of_comparisons` |
| Required n (quick approx) | `16 × p(1-p) / δ²` where δ = absolute MDE |
