# Hypothesis: Uncertainty Estimation via Deep Ensembles on Non-Stationary Fraud Data

*Committed before any modeling or results. Date: 2026-03-30*

---

## Background

This project is grounded in Lakshminarayanan, Pritzel, and Blundell (2017),
"Simple and Scalable Predictive Uncertainty Estimation using Deep Ensembles"
(NeurIPS 2017, arXiv:1612.05142). The paper proposes that training M independent
neural networks from different random initializations and aggregating their
predictions yields well-calibrated uncertainty estimates — specifically, that
the variance across ensemble members is a principled proxy for predictive
uncertainty. The authors demonstrate this on image classification and toy
regression tasks, showing that ensemble variance correctly increases on
out-of-distribution inputs and correlates with prediction error.

The original paper does not address financial tabular data, non-stationary
environments, or the operational question of whether uncertainty estimates
are useful for production decision systems. This project extends the framework
to those settings.

A secondary reference is Gorishniy et al. (2021), "Revisiting Deep Learning
Models for Tabular Data" (NeurIPS 2021, arXiv:2106.11959), which establishes
that MLPs and transformer-based architectures are competitive with gradient
boosted trees on tabular data under rigorous experimental conditions. This
motivates the use of MLP ensembles as a legitimate deep learning baseline
for structured financial data.

---

## Dataset

The IEEE-CIS Fraud Detection dataset (Kaggle, 2019) is sourced from Vesta
Corporation's real-world e-commerce transaction records. It contains 590,540
transactions spanning approximately six months, with a fraud rate of 3.5%
(20,663 fraudulent transactions). Features are split across two files:

- **train_transaction.csv**: 394 features including transaction amount,
  card metadata, time delta (TransactionDT), and anonymized features
  grouped as C, D, M, and V columns
- **train_identity.csv**: 37 features including device type, browser,
  and network metadata, joined to transactions via TransactionID

The key property that makes this dataset suitable for this study is
**TransactionDT** — a monotonically increasing time delta from a fixed
reference point. This enables construction of temporal train/test splits
that simulate production deployment conditions, where a model trained on
historical data is scored against future transactions it has never seen.
This is in contrast to the random i.i.d. splits typically used in Kaggle
competitions, which overestimate real-world generalization.

The Kaggle test set labels are not publicly available. All evaluation
therefore uses the training data with explicit temporal holdout splits
constructed from TransactionDT.

---

## Hypotheses and Research Questions

### RQ1 — Calibration
**Hypothesis:** High ensemble variance predictions correspond to actual
model errors at a higher rate than low ensemble variance predictions,
independently of the point estimate magnitude.

*In plain terms: the model's uncertainty signal is informative — when the
ensemble disagrees, it is more likely to be wrong.*

**What would falsify this:** Uncertainty-stratified accuracy curves that
are flat — i.e., high-variance predictions are no more likely to be errors
than low-variance predictions. This would indicate ensemble variance is
uninformative noise on this dataset.

**Why I expect it to hold:** The theoretical motivation from
Lakshminarayanan et al. is strong, and fraud data contains genuinely
ambiguous transactions at the boundary between fraud and non-fraud that
should produce ensemble disagreement. However, the degree to which this
holds on heavily anonymized features (the V columns) is uncertain.

---

### RQ2 — Temporal Uncertainty Drift
**Hypothesis:** Mean ensemble variance increases monotonically as
transactions are scored further in time from the training window,
reflecting the model's decreasing familiarity with evolving fraud patterns.

*In plain terms: the model knows it doesn't know — uncertainty grows
as the data drifts.*

**What would falsify this:** Flat or decreasing ensemble variance over
time windows after the training cutoff. This would suggest that either
the fraud distribution is stable over the dataset's time horizon, or
that ensemble variance is insensitive to distribution shift on this data.

**Why I expect it to hold:** Fraud patterns are known to shift as
fraudsters adapt to detection systems. The IEEE-CIS dataset spans a
period long enough to capture at least one such shift. However, because
the dataset covers only six months and features are heavily anonymized,
the effect may be weaker than in a longer production setting — this is
an acknowledged limitation upfront.

---

### RQ3 — Architectural Comparison of Uncertainty
**Hypothesis:** Deep ensemble (MLP) uncertainty and GBM ensemble
(LightGBM) uncertainty are not equivalent — they will disagree on
a non-trivial subset of transactions, and the transactions where they
disagree will be systematically interpretable (e.g., concentrated in
specific transaction types, time periods, or amount ranges).

*In plain terms: architecture affects not just what a model predicts
but how it expresses doubt — and the disagreements are meaningful.*

**What would falsify this:** High rank correlation between MLP ensemble
variance and LGBM ensemble variance across transactions, indicating
the two architectures produce essentially the same uncertainty signal
and architecture choice is irrelevant for uncertainty estimation on
this data.

**Why I expect partial support at most:** GBMs and MLPs have different
inductive biases — GBMs partition feature space via splits while MLPs
learn smooth functions. This should produce different uncertainty
surfaces, particularly on continuous features like transaction amount
and the D-columns. However, I expect the two to largely agree on
obviously fraudulent and obviously clean transactions, with divergence
concentrated in the ambiguous middle.

---

## What This Study Cannot Conclude

- Ensemble variance is not a true Bayesian posterior. It is a practical
  proxy for uncertainty, not a theoretically grounded credible interval.
- Results are specific to the IEEE-CIS dataset and may not generalize
  to other fraud domains, time horizons, or feature sets.
- The anonymized V-columns (145 features) are of unknown provenance.
  Feature-level interpretation of uncertainty is therefore limited.
- Compute constraints limit ensemble size to M=5. The sensitivity of
  results to M is not fully explored.

---

## References

Lakshminarayanan, B., Pritzel, A., & Blundell, C. (2017). Simple and
scalable predictive uncertainty estimation using deep ensembles.
Advances in Neural Information Processing Systems, 30. arXiv:1612.05142

Gorishniy, Y., Rubachev, I., Khrulkov, V., & Babenko, A. (2021).
Revisiting deep learning models for tabular data. Advances in Neural
Information Processing Systems, 34. arXiv:2106.11959
