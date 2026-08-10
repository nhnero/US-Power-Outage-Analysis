---
layout: page
title: Predicting Prolonged Power Outages
---

**Nate Nero** · [natehnero@gmail.com](mailto:natehnero@gmail.com) · [Project repository](https://github.com/nhnero/US-Power-Outage-Analysis)

<style>
.chart { position: relative; width: 100%; margin: 1.6rem 0; }
.chart iframe { width: 100%; border: 1px solid rgba(11,11,11,0.10); border-radius: 6px; display: block; }
.keynum { display: flex; flex-wrap: wrap; gap: 1.5rem; margin: 1.5rem 0 2rem; }
.keynum div { flex: 1 1 130px; }
.keynum b { display: block; font-size: 1.9rem; line-height: 1.1; color: #2a78d6; }
.keynum span { font-size: 0.85rem; color: #52514e; }
table { font-size: 0.92rem; }
</style>

When a major power outage is reported, the operational question is not *how many minutes will it
last* but a decision: **is this going to run long enough that we need to escalate?** Utilities
request mutual-aid crews from neighboring systems when restoration is expected to exceed roughly a
day, and that request has to go out early to be useful.

This project asks: **using only what is known at the moment an outage is reported, can we predict
whether it will last more than 24 hours?**

<div class="keynum">
  <div><b>0.86</b><span>ROC-AUC, random holdout</span></div>
  <div><b>0.78</b><span>ROC-AUC, 2014–16 holdout</span></div>
  <div><b>41%</b><span>base rate over 24h</span></div>
  <div><b>1,398</b><span>outages modeled</span></div>
</div>

## Introduction

The data is the DOE major-outage record compiled by Mukherjee, Nateghi, and Hastak (2018), covering
January 2000 – July 2016: **1,534 outages across 55 variables**. The DOE defines a *major* outage as
one affecting at least 50,000 customers or causing a loss of at least 300 MW of firm load.

Outages are costly and dangerous, and the burden falls hardest on customers who stay dark longest.
Knowing early which events will run long is what lets a utility pre-stage crews instead of
reacting.

## Data cleaning

Four things needed fixing before any analysis:

1. **Types.** Every numeric column arrives as `object` because of the two header rows.
2. **Placeholder zeros.** `OUTAGE.DURATION`, `DEMAND.LOSS.MW`, and `CUSTOMERS.AFFECTED` use `0` for
   "not reported." The smallest genuine duration in the data is 1 minute, and a major outage cannot
   last zero minutes, so these are missing values. Treating them as real would inflate the
   short-outage class.
3. **Timestamps.** Date and time arrive separately and are combined into real datetimes.
4. **Season.** `MONTH` is cyclical — December and January are adjacent, but as integers they are
   maximally far apart — so a season label is derived directly.

After cleaning, **1,398 outages have a usable duration.**

## Exploratory analysis

### Durations span four orders of magnitude

<div class="chart"><iframe src="{{ '/assets/duration_hist.html' | relative_url }}" height="450" frameborder="0" title="Distribution of outage durations"></iframe></div>

The median outage lasts **14.5 hours** but the mean is **46.2 hours**, and the longest ran **1,811
hours** — 75 days. A handful of extreme events drag the mean far above the typical case. That skew
drives two decisions later: medians rather than means throughout, and a threshold target rather
than a raw regression.

### Where long outages happen

<div class="chart"><iframe src="{{ '/assets/outage_map.html' | relative_url }}" height="490" frameborder="0" title="Median outage duration by state"></iframe></div>

I map the **median**, not the mean. An earlier version of this analysis mapped mean duration, which
let a single 75-day event dominate its state's color and made the map largely a picture of where
the worst outlier happened to land.

### Cause is what separates long outages from short ones

<div class="chart"><iframe src="{{ '/assets/cause_bar.html' | relative_url }}" height="450" frameborder="0" title="Share of outages over 24 hours by cause"></iframe></div>

The spread is large and intuitive. Fuel-supply emergencies and severe weather run long because they
depend on external resupply or on physically repairing damaged distribution; intentional attacks and
islanding events are typically resolved by switching and end quickly. This chart turns out to
foreshadow the entire modeling result.

### Outage size barely predicts outage length

<div class="chart"><iframe src="{{ '/assets/bi_scatter.html' | relative_url }}" height="470" frameborder="0" title="Customers affected versus duration"></iframe></div>

The log-log correlation is a modest **r = 0.35** — size accounts for roughly 12% of the variance in
length. A transmission fault can darken a million customers and be cleared by switching in under an
hour, while a rural ice storm affects far fewer people and takes a week of line work. Severity has
at least two independent dimensions, which is why this project models duration rather than reach.

## Assessing missingness

### Is `DEMAND.LOSS.MW` NMAR?

`DEMAND.LOSS.MW` is missing in **58.7%** of rows, and I believe it is **NMAR**: the probability a
utility reports megawatt loss plausibly depends on the value itself. Small losses are the ones not
worth estimating, and hard-to-estimate losses are precisely the messy, distributed ones.

What would change my mind is data about the *reporting process* rather than the outage — which
utilities had advanced metering in a given year, when requirements changed, whether a utility
habitually substituted customers-affected for demand loss. With those columns the missingness would
likely become MAR.

### Missingness depends on cause, not on month

Comparing the cause distribution of rows where demand loss is reported against rows where it is not,
using **total variation distance** (TVD = ½·Σ\|p − q\|, the correct statistic for two categorical
distributions):

| Test | Statistic | p-value | Conclusion |
|---|---|---|---|
| Missingness vs. `CAUSE.CATEGORY` | TVD = 0.441 | < 1/5000 | Dependent |
| Missingness vs. `MONTH` | \|Δ mean\| = 0.200 | 0.237 | No evidence of dependence |

<div class="chart"><iframe src="{{ '/assets/missing_dependence_perm.html' | relative_url }}" height="450" frameborder="0" title="Permutation distribution for missingness dependence"></iframe></div>

No permuted dataset out of 5,000 reached the observed TVD, so the p-value is reported as **< 1/5000**
rather than 0 — a permutation test cannot resolve a p-value below one over the number of shuffles.
The missingness is structured by *what kind of event it was*, not by *when it happened*, consistent
with the NMAR argument.

## Hypothesis test: does cause matter for duration?

- **Null:** duration has the same distribution across all cause categories.
- **Alternative:** at least one cause differs.
- **Statistic:** sum of absolute deviations of group median `log10(duration)` from the overall
  median — medians and a log scale because of the skew documented above.
- **Test:** permutation test, 5,000 shuffles, α = 0.05.

The observed statistic is **4.61**, far outside the null distribution, giving **p < 1/5000**.
Outage duration is associated with cause category.

This is an association, not a causal claim: cause is entangled with geography and season, so the
test says cause *carries information about* duration, not that it *produces* it.

## Framing the prediction problem

**Task.** Binary classification — will an outage exceed **24 hours**, predicted at the moment it is
reported?

**Why a threshold, not a regression.** The decision it supports is discrete — escalate to mutual aid
or handle it locally — and must be made early. A calibrated probability of crossing one meaningful
line is more useful than a point estimate of minutes with an error bar wider than the estimate.

**Why 24 hours.** It is the rough industry threshold for requesting mutual-aid crews, and **40.8%**
of outages exceed it, so neither class is rare. It was chosen for operational meaning, not fitted to
the data. An earlier version of this project used quantile bins, which produced a "medium" class
defined only as the gap between two cutoffs that no model could learn.

**Metric.** ROC-AUC as the headline — it evaluates the ranking of risk across all thresholds and is
insensitive to the class balance shifting between the random and chronological splits. Average
precision, calibration, and precision/recall at the deployed threshold are reported alongside it.

**The information constraint.** Every feature must be knowable when the outage is *reported*. Two
columns used in the earlier version of this project are excluded for that reason:

| Column | Why excluded |
|---|---|
| `CUSTOMERS.AFFECTED` | Reported as the event unfolds and revised afterward; not available at hour zero. |
| `DEMAND.LOSS.MW` | A post-event reported quantity, and NMAR as shown above — its very presence in a row carries outcome information. |

Their missingness indicators are dropped too: whether a utility eventually filed a demand-loss figure
is a fact about the aftermath. Including these inflates the score while making the model useless for
the decision it claims to support.

Everything that learns from data — imputation, scaling, one-hot vocabularies, and the grouping of
rare `CAUSE.CATEGORY.DETAIL` levels — happens **inside a scikit-learn `Pipeline`**, so it is fit on
training folds only.

## Baseline and model selection

The baseline is a **regularized logistic regression** on the same features and pipeline — a
deliberately strong baseline, since an under-tuned tree makes any successor look good and tells you
nothing. Three families were tuned by `GridSearchCV` on cross-validated ROC-AUC **over the training
set only**; the test set was not consulted until one model had been chosen.

| Model | CV ROC-AUC | Best parameters |
|---|---|---|
| **Random forest (final)** | **0.8581** | `max_depth=8, max_features=0.4, min_samples_leaf=1, n_estimators=300` |
| Gradient boosting | 0.8444 | `learning_rate=0.03, max_leaf_nodes=8, min_samples_leaf=10` |
| Logistic regression (baseline) | 0.8434 | L2 regularized |

**Read this honestly:** the spread across all three families is about 0.015 AUC against a
cross-validation standard deviation of **±0.030**. These models are statistically indistinguishable.
The random forest is selected because it scored best, but the accurate statement is that a plain
logistic regression is as good as anything else here. I also tested adding state, per-capita GSP,
utility contribution, population, and land-area features; every variant landed inside the same noise
band, so the parsimonious feature set was kept.

### Results on the random holdout

| | Baseline (logistic) | Final (random forest) |
|---|---|---|
| ROC-AUC | 0.8560 | **0.8596** |
| Average precision | 0.8009 | 0.7923 |

<div class="chart"><iframe src="{{ '/assets/roc_curve.html' | relative_url }}" height="490" frameborder="0" title="ROC curves"></iframe></div>

### Calibration and the operating point

<div class="chart"><iframe src="{{ '/assets/calibration.html' | relative_url }}" height="470" frameborder="0" title="Calibration curve"></iframe></div>

Predicted probabilities track observed frequencies closely, so "70% chance of running long" means
roughly what it says. At an operating threshold of **0.515** — the highest-recall point that keeps
precision at or above 0.70, reflecting that a missed long outage costs more than an unnecessary crew
alert — the model achieves **precision 0.702 and recall 0.763**, catching about three-quarters of
the outages that truly ran long.

### One feature carries nearly all the signal

<div class="chart"><iframe src="{{ '/assets/importance.html' | relative_url }}" height="490" frameborder="0" title="Permutation importance"></iframe></div>

Permutation importance measured on the test set, so it reports what generalizes:

| Feature | Drop in test ROC-AUC when shuffled |
|---|---|
| `CAUSE.CATEGORY` | **0.2480** |
| `POPDEN_RURAL` | 0.0112 |
| `CAUSE.CATEGORY.DETAIL` | 0.0090 |
| `CLIMATE.REGION` | 0.0053 |
| `POPPCT_URBAN` | 0.0036 |

Shuffling cause category costs **0.25 of AUC; nothing else costs more than 0.01.** Cause is not
merely the strongest predictor — it is very nearly the only one.

This explains the rest of the project at once. It is why the model families tie: there is one
dominant, largely categorical signal and every family captures it. It is why extra covariates
changed nothing. And it sets a realistic ceiling — without a feature that distinguishes *severity
within* a cause (storm intensity, damage assessments, crew availability), more modeling will not
help. That, not algorithm choice, is where a real improvement would come from.

## The stricter test: chronological validation

A random split lets the model train on 2015 and predict 2009. Nothing deployed can do that. The
honest test trains on **2000–2013** and predicts **2014–2016**.

| | Random split | Chronological split |
|---|---|---|
| ROC-AUC | 0.8596 | **0.7790** |
| Average precision | 0.7923 | 0.4800 |
| Positive rate, test | 40.8% | 24.4% |

The share of outages exceeding 24 hours **falls from 44.4% in the training era to 24.4% in the test
era**. The model is predicting a different world from the one it learned. Average precision falls
much further than AUC — the expected signature, since average precision depends on the base rate and
AUC does not. Ranking ability largely survives; absolute probabilities do not, and would need
recalibration on recent data before deployment.

Plausible drivers include improved restoration practice and grid automation over the period, and
changes in DOE reporting practice affecting which events enter the dataset at all. This data cannot
distinguish them.

**The chronological number is the one I would quote** as an estimate of deployed performance. The
random-split figure is reported for comparability, not because it is the more trustworthy of the two.

## Secondary question: how long, exactly?

A gradient-boosted regressor on `log10(duration)` — logged because being wrong by 5 hours matters far
more on a 6-hour outage than on a 600-hour one.

| Metric | Value |
|---|---|
| R² on log10 duration | 0.565 |
| RMSE (log10 hours) | 0.707 |
| Median error | **2.57×** (predict-the-median baseline: 4.06×) |

<div class="chart"><iframe src="{{ '/assets/residuals.html' | relative_url }}" height="460" frameborder="0" title="Regression residuals"></iframe></div>

A typical prediction is off by a factor of 2.6 — better than the baseline's 4.1, but wide enough that
quoting a specific restoration time to a customer would be misleading. The residuals show the classic
shape: the model regresses toward the middle, over-predicting the shortest outages and
under-predicting the longest.

This is the concrete argument for the threshold framing. The same features that cannot pin down *how
long* an outage will last can still say, with useful accuracy, *whether it will cross a line that
matters*.

## Fairness analysis

Does the model serve warm- and cold-climate regions equally well? If it ranks risk reliably in one
and poorly in the other, deploying it would systematically under-resource the weaker group.

- **Group X — warm:** South, Southeast, Southwest (n = 97)
- **Group Y — cold:** Northeast, Northern Rockies & Plains, Upper Midwest (n = 62)

Both groups are defined **explicitly**, and rows outside them are excluded rather than swept into one
side by a default — an earlier version used "warm vs. everything else," which silently placed missing
regions in the cold group.

- **Metric:** ROC-AUC. **Statistic:** AUC(warm) − AUC(cold).
- **Null:** the model ranks risk equally well in both groups.
- **Test:** two-sided permutation test, 2,000 shuffles, α = 0.05.

| Group | ROC-AUC |
|---|---|
| Warm | 0.7865 |
| Cold | 0.8250 |
| **Difference** | **−0.0385** (p = 0.579) |

<div class="chart"><iframe src="{{ '/assets/fairness_perm.html' | relative_url }}" height="450" frameborder="0" title="Fairness permutation distribution"></iframe></div>

The observed gap sits well inside the null distribution, so at α = 0.05 there is **no evidence the
model performs differently** across warm and cold climate regions.

Two caveats. "Fail to reject" is not proof of fairness — with roughly 60–100 test rows per group this
test only detects fairly large gaps. And climate region is one grouping among several; a rural/urban
or investor-owned/municipal split could behave differently and is not tested here.

## Limitations

- **Sample size.** 1,398 usable outages over sixteen years. The ±0.03 cross-validation standard
  deviation is the binding constraint on every comparison here, and is why model families are
  reported as indistinguishable rather than ranked.
- **Survivorship in the target.** Rows with no recorded duration are dropped. If reporting relates to
  severity, the modeled population is not the full population of major outages.
- **The 24-hour line is a choice.** A utility with different mutual-aid economics would want a
  different threshold; the useful artifact is the calibrated probability, from which any threshold
  can be derived.
- **Distribution shift is real and unexplained.** Measurable here, not attributable with this data.
- **Association, not causation.** Nothing here supports intervening on a feature to change an outcome.

## Conclusion

Using only information available when an outage is reported, whether it will run past 24 hours is
**predictable well enough to be useful** — ROC-AUC 0.86 on a random holdout and 0.78 on a
chronological one, against a 41% base rate. Almost all of that signal comes from a single feature,
cause category, and a logistic regression captures nearly all of it.

The more useful findings are the negative ones. Outage size barely predicts outage length. One
feature does nearly all the work, and neither extra covariates nor extra model capacity added
anything measurable. And the gap between the random and chronological splits is several times larger
than the gap between any two model families — which says that for this problem, **how you validate
matters more than what you fit.**
