# Marketing A/B Testing: Ad vs. PSA

A rigorous **A/B testing and statistical analysis** project to determine whether showing commercial advertisements instead of Public Service Announcements (PSAs) improves user conversion.

The analysis goes beyond simply checking whether a result has `p < 0.05`. It evaluates **randomization quality, statistical power, effect size, confidence intervals, bootstrap robustness, segment-level effects, interaction effects, and practical business significance**.

---

## 🎯 Business Problem

The key business question is:

> **Should commercial ads be shown instead of PSAs to the full user base?**

The experiment compares two groups:

* **Ad group** — users exposed to commercial advertisements
* **PSA group** — users exposed to public service announcements

### Primary Metric

**Conversion Rate (CR)**

```text
Conversion Rate = Number of Converted Users / Total Users
```

### Practical Business Threshold

Before looking at the experimental results, the project defines a **Minimum Detectable / Meaningful Effect of 0.5 percentage points**.

Therefore, statistical significance alone is **not sufficient** for the final recommendation.

The observed improvement must also be large enough to clear the predefined **0.5 pp practical threshold**.

---

# 📊 Dataset

The project uses the **Marketing A/B Testing dataset from Kaggle**.

Dataset source:

**Marketing A/B Testing — Kaggle**

The notebook expects the downloaded dataset at:

```text
data/marketing_AB.csv
```

The data contains information including:

* `user_id`
* `test_group`
* `converted`
* `total_ads`
* `most_ads_day`
* `most_ads_hour`

The dataset is loaded using Pandas and cleaned before analysis.

---

# 🔬 Analysis Workflow

```text
                    Marketing A/B Dataset
                            │
                            ▼
                    Data Cleaning
                            │
                            ▼
                    Data Quality Checks
                            │
                            ▼
                  Randomization Check
                            │
                            ▼
                    Power Analysis
                            │
                            ▼
                  Exploratory Analysis
                            │
                            ▼
               Main Hypothesis Test
                            │
                            ▼
                Confidence Intervals
                            │
                            ▼
                    Bootstrap Test
                            │
                            ▼
                     Segmentation
                      /         \
                     ▼           ▼
              Day of Week    Engagement
                     \           /
                      ▼         ▼
                  Interaction Tests
                            │
                            ▼
                    Business Decision
```

---

# 🧹 1. Data Loading & Cleaning

The notebook first loads the CSV using Pandas.

```python
df = pd.read_csv("data/marketing_AB.csv")
```

It then performs several cleaning operations:

### Remove unnecessary index

If the Kaggle export contains `Unnamed: 0`, it is removed.

### Rename columns

The original column names are converted into cleaner Python-friendly names:

```text
user id        → user_id
test group     → test_group
total ads      → total_ads
most ads day   → most_ads_day
most ads hour  → most_ads_hour
```

### Data types

Categorical variables such as `test_group` and `most_ads_day` are converted to categorical types, while `converted` is explicitly converted to integer.

---

# ✅ 2. Data Quality Checks

Before performing statistical analysis, the project checks the quality of the dataset.

The following are inspected:

* Full-row duplicates
* Duplicate `user_id`s
* Missing values
* Unique values of `converted`
* Group sizes
* Conversion rate by experimental group

```python
df.duplicated().sum()
df.duplicated("user_id").sum()
df.isna().sum()
```

The conversion rate and sample size are also calculated separately for the Ad and PSA groups.

---

# 🎲 3. Randomization / Balance Check

Before interpreting the treatment effect, the project checks whether the two experimental groups are reasonably comparable.

The analysis examines whether the distribution of:

* Day of week
* Hour of day

differs between the Ad and PSA groups.

Chi-square tests are used:

```python
chi2_contingency(contingency_day)
chi2_contingency(contingency_hour)
```

### Why is this important?

If the treatment groups have substantially different exposure patterns, the observed conversion difference could potentially be influenced by factors other than the treatment itself.

However, because the dataset contains roughly **588K observations**, even tiny differences can become statistically significant.

Therefore, the project does **not rely only on the chi-square p-value**.

It also examines the actual distribution proportions and visualizes the group shapes.

---

# 📐 4. Statistical Power Analysis

The project performs **prospective power analysis** before interpreting the main experiment.

The predefined business threshold is:

```text
Minimum meaningful lift = 0.5 percentage points
```

The analysis asks:

1. What sample size per group is required to detect a 0.5 pp improvement?
2. Does the actual experiment have enough observations?

The baseline is the PSA conversion rate:

```python
cr_psa = df.loc[
    df["test_group"] == "psa",
    "converted"
].mean()
```

The expected treatment conversion rate under the MDE is:

```python
cr_mde = cr_psa + 0.005
```

The analysis targets:

```text
α = 0.05
Power = 80%
Alternative = one-sided
```

The required sample size is calculated using `statsmodels`.

### Why power analysis?

A non-significant result can mean two very different things:

```text
No meaningful effect exists
          OR
The experiment does not have enough data
```

Power analysis helps distinguish between these situations.

---

# 📈 5. Exploratory Data Analysis

The project explores conversion behavior across different dimensions.

### Conversion by day

Conversion rate is calculated for each:

```text
Day × Experimental Group
```

This allows comparison of Ad vs. PSA performance throughout the week.

### Conversion by hour

The project also examines conversion rates by hour for the Ad group.

These analyses help identify patterns and provide context before performing the formal hypothesis test.

---

# 🧪 6. Main Hypothesis Test

The primary statistical test compares conversion rates between the two groups.

### Null Hypothesis

```text
H₀: CR(Ad) = CR(PSA)
```

### Alternative Hypothesis

```text
H₁: CR(Ad) > CR(PSA)
```

The alternative is **one-sided** because the business question is specifically whether commercial advertising improves conversion.

---

## Two-Proportion Z-Test

Because `converted` is a binary variable, the project uses a **two-proportion z-test**.

It does **not** use a t-test because a t-test is designed for comparing means of continuous variables.

```python
z_stat, p_val = proportions_ztest(
    conv,
    n_obs,
    alternative="larger"
)
```

---

# 📏 7. Effect Size

Statistical significance does not tell us how large the treatment effect is.

Therefore, the project calculates both:

### Absolute Lift

```text
Ad Conversion Rate - PSA Conversion Rate
```

reported in **percentage points**.

### Relative Lift

```text
(Ad CR - PSA CR) / PSA CR
```

reported as a percentage.

This allows the analysis to answer both:

> Is there evidence of an effect?

and:

> How large is the effect?

The notebook explicitly compares the confidence interval against the predefined **0.5 pp business threshold**.

---

# 📊 8. Confidence Interval

A 95% confidence interval is calculated for the difference in conversion rates.

```python
confint_proportions_2indep(
    conv["ad"],
    n_obs["ad"],
    conv["psa"],
    n_obs["psa"]
)
```

The confidence interval provides a range of plausible values for the actual treatment effect.

### Business interpretation

The key question is not merely:

```text
Does the CI exclude 0?
```

It is:

```text
Does the lower confidence bound exceed +0.5 percentage points?
```

If it does, the evidence supports an improvement large enough to cross the predefined practical threshold.

---

# 🔁 9. Bootstrap Robustness Check

The project performs a **non-parametric bootstrap** as a robustness check for the analytical z-test and confidence interval.

The procedure:

1. Separate Ad and PSA conversion outcomes.
2. Resample each group with replacement.
3. Calculate the difference in conversion rates.
4. Repeat 2,000 times.
5. Construct the bootstrap 95% confidence interval.

```python
n_boot = 2000
```

The bootstrap interval is then compared with the analytical confidence interval.

### Why bootstrap?

It provides an independent, distribution-light validation of the estimated treatment effect.

If both approaches tell a similar story, confidence in the result increases.

---

# 👥 10. Segmentation Analysis

The project investigates whether the treatment effect is consistent across different user segments.

Two segmentation dimensions are analyzed:

### 1. Day of Week

The treatment effect is evaluated separately for:

```text
Monday
Tuesday
Wednesday
Thursday
Friday
Saturday
Sunday
```

### 2. Engagement Tier

The dataset does not contain an explicit:

```text
New User / Returning User
```

variable.

Therefore, `total_ads` is used as a **proxy for user engagement/tenure**.

Users are divided into three tertiles:

```text
Low engagement
Medium engagement
High engagement
```

This is explicitly treated as a modeling choice rather than a ground-truth user-tenure label.

---

# 🧮 11. Segment-Level Hypothesis Tests

For each segment, the project calculates:

* p-value
* FDR-adjusted p-value
* Absolute lift
* Confidence interval
* Statistical significance after multiple-testing correction

For day-level analysis, the project uses **Benjamini-Hochberg False Discovery Rate (FDR) correction**.

```python
multipletests(
    pvals,
    method="fdr_bh"
)
```

The same correction is applied to the engagement-tier comparisons.

### Why multiple-testing correction?

When many hypotheses are tested simultaneously, the probability of obtaining at least one false positive increases.

FDR correction controls the expected proportion of false discoveries among the rejected hypotheses.

---

# 🔀 12. Interaction Effects

A particularly important part of the analysis is testing whether the treatment effect **actually differs between segments**.

A common mistake is:

> "The result is significant in Segment A but not significant in Segment B, therefore the treatment works differently."

That conclusion is not necessarily valid.

Instead, the project explicitly tests the **interaction between treatment and segment**.

---

## Day Interaction

Two approaches are used:

### Linear Probability Model / ANOVA

```python
converted ~ C(test_group) * C(most_ads_day)
```

### Logistic Regression

```python
converted ~ C(test_group) * C(most_ads_day)
```

The interaction coefficients and p-values are examined.

---

## Engagement Interaction

A logistic regression interaction model is also fitted:

```python
converted ~ C(test_group) * C(engagement_tier)
```

This tests whether the effect of advertising changes across low-, medium-, and high-engagement users.

### Correct interpretation

A significant interaction means:

> The treatment effect itself differs between segments.

Simply finding that one segment has `p < 0.05` while another has `p > 0.05` is **not sufficient evidence** that the treatment effects differ.

---

# 📋 Statistical Framework

| Analysis                | Purpose                                            |
| ----------------------- | -------------------------------------------------- |
| Data quality checks     | Ensure data reliability                            |
| Chi-square test         | Check group balance                                |
| Power analysis          | Determine whether experiment is adequately powered |
| Two-proportion z-test   | Test primary treatment effect                      |
| Effect size             | Quantify practical impact                          |
| 95% CI                  | Quantify uncertainty                               |
| Bootstrap               | Robustness check                                   |
| Segment tests           | Identify potential heterogeneous effects           |
| FDR correction          | Control multiple-testing false discoveries         |
| ANOVA / OLS interaction | Test treatment × segment interaction               |
| Logistic interaction    | Binary-outcome interaction cross-check             |

---

# 🛠️ Tech Stack

### Programming

* Python

### Data Analysis

* Pandas
* NumPy

### Visualization

* Matplotlib

### Statistical Analysis

* SciPy
* Statsmodels

### Statistical Methods

* Chi-square test
* Two-proportion z-test
* Power analysis
* Confidence intervals
* Bootstrap
* Multiple-testing correction
* ANOVA
* Linear Probability Model
* Logistic Regression
* Interaction analysis

---

# 📁 Project Structure

```text
marketing-ab-testing/
│
├── data/
│   └── marketing_AB.csv
│
├── marketing_ab_testing_full.ipynb
│
└── README.md
```

---

# 🚀 Getting Started

## 1. Clone the repository

```bash
git clone <your-repository-url>
cd marketing-ab-testing
```

## 2. Install dependencies

```bash
pip install numpy pandas matplotlib scipy statsmodels jupyter
```

## 3. Add the dataset

Download the Marketing A/B Testing dataset from Kaggle and place the CSV at:

```text
data/marketing_AB.csv
```

## 4. Launch Jupyter Notebook

```bash
jupyter notebook
```

Open:

```text
marketing_ab_testing_full.ipynb
```

## 5. Run the notebook

Execute the cells sequentially to reproduce:

```text
Data Cleaning
      ↓
Data Quality
      ↓
Randomization Check
      ↓
Power Analysis
      ↓
EDA
      ↓
Hypothesis Testing
      ↓
Confidence Intervals
      ↓
Bootstrap
      ↓
Segmentation
      ↓
Interaction Analysis
```

---

# 💡 Key Analytical Decisions

### Why use a two-proportion z-test?

Because the outcome variable `converted` is binary:

```text
0 = Not converted
1 = Converted
```

The objective is to compare two population proportions.

---

### Why use a one-sided test?

The business question is directional:

> Does showing ads produce a **higher** conversion rate than PSAs?

Therefore:

```text
H₁: CR(Ad) > CR(PSA)
```

is used rather than a two-sided alternative.

---

### Why define the 0.5 pp threshold beforehand?

Without a predefined practical threshold, it is easy to treat a statistically significant but commercially negligible effect as a successful experiment.

The project therefore separates:

```text
Statistical significance
        +
Practical significance
        ↓
Business decision
```

The 0.5 pp threshold is established before interpreting the results.

---

### Why use bootstrap?

The bootstrap provides a non-parametric cross-check of the analytical confidence interval and treatment-effect estimate.

---

### Why perform interaction tests?

Because separate segment p-values do not prove that treatment effects differ.

Interaction terms directly test:

```text
Does treatment effect × segment
produce a statistically meaningful interaction?
```

---

# ⚠️ Limitations

### 1. No guardrail metric

The dataset does not contain cost or spend information.

Therefore, the analysis cannot evaluate whether a higher conversion rate comes at an unacceptable advertising cost.

### 2. No native new/returning-user variable

The dataset does not directly identify whether a user is new or returning.

`total_ads` exposure is therefore used as a proxy for engagement/tenure.

This should not be interpreted as a ground-truth tenure classification.

### 3. Imbalanced treatment groups

The dataset contains approximately:

```text
96% Ad
4% PSA
```

This imbalance is particularly relevant for segment-level analyses because smaller groups have less statistical power.

### 4. Results depend on the actual downloaded dataset

The notebook's conclusion section is structured to be populated after executing the analysis against the actual CSV.

Therefore, this README intentionally does **not invent final conversion rates, p-values, confidence intervals, or a ship/no-ship recommendation** that are not present as executed outputs in the supplied notebook.

---

# 📌 Final Decision Framework

The final recommendation should follow this logic:

```text
                 Is the result statistically significant?
                              │
                    ┌─────────┴─────────┐
                   NO                   YES
                    │                    │
             Don't ship /        Does 95% CI
             gather data        clear +0.5 pp?
                                         │
                               ┌─────────┴─────────┐
                              NO                  YES
                               │                    │
                       Effect may be          Evidence supports
                       statistically          a practically
                       significant but        meaningful lift
                       too small              → Consider shipping
```

The central principle of the project is:

> **Do not make the business decision based solely on p-value. Evaluate statistical evidence, uncertainty, effect size, practical significance, and segment behavior together.**

---

# 🎤 Interview-Ready Summary

### One-line explanation

> **Performed a rigorous A/B test comparing commercial ads with PSAs, using power analysis, two-proportion hypothesis testing, confidence intervals, bootstrap validation, FDR-corrected segmentation, and treatment-segment interaction models to distinguish statistical from practical significance.**

### 30-second explanation

> I analyzed a marketing A/B testing experiment where users were assigned to either commercial ads or PSAs, with conversion rate as the primary metric. Instead of only checking the p-value, I first checked randomization balance and whether the experiment was sufficiently powered to detect a predefined 0.5 percentage-point improvement. I then used a one-sided two-proportion z-test, calculated the absolute and relative lift with confidence intervals, and validated the result using bootstrap resampling. Finally, I analyzed heterogeneous treatment effects by day and engagement level, applying FDR correction and formal interaction tests to determine whether the treatment effect genuinely varied across segments.

### Most important takeaway

> **Statistical significance tells us whether the evidence is inconsistent with no effect; practical significance tells us whether the effect is large enough to matter to the business. This project explicitly evaluates both.**
