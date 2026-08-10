# Predicting Prolonged Power Outages

**[→ Read the full analysis](https://nhnero.github.io/US-Power-Outage-Analysis/)**

When a major power outage is reported, utilities need to decide early whether to request mutual-aid
crews from neighboring systems. This project asks whether that call can be made at hour zero:
**using only information available when an outage is reported, can we predict whether it will last
more than 24 hours?**

Data is the DOE major-outage record compiled by Mukherjee, Nateghi, and Hastak (2018) — 1,534 U.S.
outages from January 2000 to July 2016.

## Results

| | |
|---|---|
| ROC-AUC, random holdout | **0.860** |
| ROC-AUC, chronological holdout (train ≤2013, test 2014–16) | **0.779** |
| Base rate (outages over 24h) | 40.8% |
| Operating point | precision 0.702 / recall 0.763 at threshold 0.515 |
| Final model | Random forest, 13 report-time features |

Three findings worth more than the headline number:

- **One feature does nearly all the work.** Shuffling `CAUSE.CATEGORY` costs 0.25 of test AUC;
  no other feature costs more than 0.01.
- **Model choice barely matters.** Logistic regression, random forest, and gradient boosting land
  within 0.015 AUC of each other against a ±0.030 cross-validation standard deviation — they are
  statistically indistinguishable.
- **Validation choice matters a lot.** The random-to-chronological gap is several times larger than
  any gap between model families, driven by the over-24h rate falling from 44% to 24% across the
  period.

## Methods

- Permutation tests for missingness dependence (TVD) and for duration-vs-cause association.
- NMAR argument for `DEMAND.LOSS.MW`, which is also excluded from the model as post-event
  information — along with `CUSTOMERS.AFFECTED` — so the feature set respects the "known at report
  time" constraint.
- All learned transformations (imputation, scaling, one-hot vocabularies, rare-level grouping) live
  inside a scikit-learn `Pipeline` so they are fit on training folds only.
- Both a stratified random split and a chronological split, plus a calibration curve, permutation
  importance, a secondary log-duration regression, and a two-sided fairness permutation test across
  warm and cold climate regions.

## Repository

```
outage_analysis.ipynb   full analysis, executed end to end
index.md                the project write-up rendered at the site above
results.json            every number quoted in the write-up, emitted by the notebook
data/outage.xlsx        source dataset
assets/*.html           interactive Plotly figures embedded in the site
```

## Running it

```bash
conda create -n outage python=3.12 pandas scikit-learn plotly openpyxl jupyter
conda activate outage
jupyter lab outage_analysis.ipynb
```

Restart-and-run-all reproduces every figure and rewrites `results.json`. Built with
scikit-learn 1.5.2, pandas 2.2.3, and plotly 5.24.1.

---

Nate Nero · [natehnero@gmail.com](mailto:natehnero@gmail.com) · originally coursework for DSC 80 at UCSD,
since rebuilt.
