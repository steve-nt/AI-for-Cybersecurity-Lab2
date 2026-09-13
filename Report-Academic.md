# Learning When Labels Are Scarce: Semi-Supervised Intrusion Detection on CICIDS2017

**Lab 2, D7084E/D7041E.** Companion plain-language version: `Report-Plain-Language.md`.
All results come from `Lab_2_SSL_CICIDS_FIXED.ipynb` executed end to end on Google Colab
(Python 3.13.15, scikit-learn 1.6.1) on the verified Lab 1 split, seeds 42/43/44. The notebook's
section 14 automated checks passed.

---

## 1. Why labels are scarce, and what semi-supervised learning is

Supervised intrusion detection assumes a labelled corpus of network flows. In operational settings
that assumption fails: labelling a flow requires an analyst to reconstruct and judge it, so labels
are produced at human speed while traffic accumulates at machine speed. The practical situation is a
small labelled set alongside a far larger unlabelled one.

Semi-supervised learning (SSL) exploits the unlabelled portion. This report evaluates two SSL
methods against the two baselines that make such a claim falsifiable: a model trained only on the
scarce labels (the lower bound SSL must beat) and the full-label Lab 1 detector (the upper bound it
tries to approach). Reporting either method without both baselines would be uninterpretable.

## 2. Experimental setup

**Data and reuse of Lab 1.** CICIDS2017 (Sharafaldin et al., 2018), cleaned by the Lab 1 pipeline:
identifier and leakage-prone columns dropped, infinities and missing values removed, duplicates
removed, and a stratified 20% sample retained, giving 446,641 flows and 68 numeric features at a
15.07% attack rate. Lab 1's own 60/20/20 stratified partition is reused **unchanged**:
267,984 training, 89,328 validation and 89,329 test flows. The task is binary (0 = benign,
1 = attack).

Reuse is enforced mechanically rather than assumed. The notebook accepts only an exported copy of
Lab 1's split and aborts unless the three shapes, the test class counts (75,868 benign / 13,461
attack) and the overall attack rate match Lab 1 exactly. It also computes a SHA-256 digest of the
arrays; the Colab run reproduced the digest of the original export
(`1891044e6bb39ea93549cd7b80f70cd83801efbbe59c69280a0ed90230d8f5ac`), so the data used here is
byte-identical to Lab 1's split. Lab 1's `false_alarm_rate` is imported from Lab 1's `metrics.py` and
cross-checked against an independent confusion-matrix computation at every evaluation.

**Label budgets.** From the training set only, a stratified subset of 1%, 5% or 10% retains its
labels (2,680 / 13,399 / 26,798 flows); the remaining 265,304 / 254,585 / 241,186 flows form the
unlabelled pool. Validation and test labels are never hidden and never used for fitting. Each
configuration is repeated with seeds 42, 43 and 44 and averaged; at the 1% budget the identity of
the retained rows is itself a source of variance (Section 5).

**Base learner.** One model is used for every method, so differences are attributable to the method
and not the estimator: scikit-learn `HistGradientBoostingClassifier` (learning rate 0.08, ≤120
iterations, 31 leaves, L2 = 1.0, early stopping on an internal 10% split), with class-balanced
sample weights. The backend is pinned and numerical libraries are limited to 4 threads.

**Metrics.** Accuracy, macro-F1, attack recall, ROC-AUC and false alarm rate (FAR = FP/(FP+TN)).
With 84.9% benign traffic, accuracy is reported only for completeness; macro-F1 and FAR carry the
argument, with recall as the operational safety check. The test set is evaluated once per trained
model, after all design decisions are fixed; the ablation uses the validation set only.

## 3. Methods

**Pseudo-labelling (required).** Self-training in the sense of Lee (2013). The model trained on the
labelled slice predicts the unlabelled pool; predictions at or above a confidence threshold of 0.95
are adopted as labels and the model is refitted, for 2 rounds. Adopted rows carry half the weight of
genuine labels. A per-class cap of `⌈n_labelled/2⌉` rows per round, taking the most confident rows
first, limits how many predictions are adopted. The true labels of adopted rows are read **after**
selection only, to audit precision; that audit never influences selection, fitting or stopping.

**Co-training (second method).** Following Blum & Mitchell (1998), the 68 features are split by a
seeded permutation into two disjoint views of 34. A learner is trained per view; each passes its
confident predictions (same threshold and cap) to the *other* learner, and rows on which the two
disagree are rejected. Final predictions average the two views' probabilities. Co-training was chosen
over Mean Teacher and FixMatch because it needs no neural network and, more importantly, no data
augmentation: there is no established semantics-preserving perturbation for tabular flow records,
which makes consistency-based methods hard to justify here.

## 4. Results

**Table 1.** Test-set performance by label budget. Lab 2 rows are means over seeds 42/43/44 with the
sample standard deviation of macro-F1; Δ is each SSL method's mean per-seed macro-F1 gain over the
same-seed, same-budget lower baseline. The final row is the original Lab 1 Random Forest, not a
retrained substitute.

| Budget | Method | Accuracy | Macro-F1 | Δ vs lower | Recall | ROC-AUC | FAR |
|---|---|---|---|---|---|---|---|
| 1% | Few-label lower baseline | 0.9903 | 0.9809 ± 0.0012 | — | 0.9569 | 0.9947 | 0.0038 |
| 1% | Pseudo-labelling | 0.9917 | 0.9836 ± 0.0032 | **+0.0027** | 0.9581 | 0.9959 | 0.0023 |
| 1% | Co-training | 0.9919 | **0.9839 ± 0.0039** | **+0.0031** | 0.9572 | 0.9975 | **0.0020** |
| 5% | Few-label lower baseline | 0.9970 | **0.9941 ± 0.0003** | — | 0.9896 | 0.9996 | 0.0017 |
| 5% | Pseudo-labelling | 0.9969 | 0.9938 ± 0.0008 | −0.0003 | 0.9861 | 0.9994 | 0.0012 |
| 5% | Co-training | 0.9969 | 0.9939 ± 0.0007 | −0.0002 | 0.9840 | 0.9994 | **0.0008** |
| 10% | Few-label lower baseline | 0.9971 | 0.9943 ± 0.0004 | — | 0.9911 | 0.9997 | 0.0019 |
| 10% | Pseudo-labelling | 0.9974 | **0.9950 ± 0.0005** | +0.0007 | 0.9900 | 0.9998 | 0.0012 |
| 10% | Co-training | 0.9974 | 0.9949 ± 0.0006 | +0.0006 | 0.9872 | 0.9997 | **0.0008** |
| 100% | Full-label same-model ceiling | 0.9980 | 0.9961 ± 0.0002 | — | 0.9986 | 1.0000 | 0.0021 |
| 100% | **Lab 1 full-label Random Forest** | 0.9984 | 0.9968 | — | 0.9923 | 0.9999 | 0.0006 |

**Figure 1** (`lab2_outputs/macro_f1_vs_label_budget.png`) plots mean test macro-F1 against the
labelled fraction with one-standard-deviation bands, the Lab 1 Random Forest as a dashed line (0.9968)
and the same-model ceiling as a dotted line (0.9961). At 1% the pseudo-labelling and co-training
curves lie on top of each other, about 0.003 above the lower baseline, and the three shaded bands
overlap. By 5% all three curves have converged and remain indistinguishable at 10%, just below the
two reference lines, which themselves nearly coincide. Note that the y-axis spans roughly 0.90–1.01,
so the differences discussed below occupy only a few percent of the plot's height.

**Pseudo-label quality and the role of the cap.** Adopted labels were essentially always correct:
precision 0.9993 / 1.0000 / 1.0000 for pseudo-labelling at 1% / 5% / 10%, and 0.9996 / 1.0000 /
0.9999 for co-training. Adoption volumes were 5,360 / 26,800 / 53,596 rows for pseudo-labelling and
7,797 / 39,864 / 75,695 for co-training. The diagnostics show that the threshold was never the binding
constraint in these runs: 94–99% of the remaining pool cleared 0.95 in every round, and the cap then
kept only 1.0–1.1% of those rows at the 1% budget (5.3–5.6% at 5%, 11.2–12.6% at 10%). The adopted
rows were therefore the top roughly 1% most-confident predictions, which explains the near-perfect
precision.

**Table 2.** Threshold ablation at the 1% budget, seed 42, scored on **validation** (the test set is
not touched). Only the confidence threshold changes between rows. The per-class cap is disabled for
every "after" row, so adoption is governed by the threshold alone; `passed = adopted` in every row
confirms this. Precision is measured against the hidden labels after selection, for analysis only.
The unlabelled pool contains 265,304 flows.

| Setting | Threshold | Passed = adopted | Share of pool | Pseudo-label precision | Macro-F1 | Δ | Recall | FAR |
|---|---|---|---|---|---|---|---|---|
| Before: few-label baseline | — | 0 | 0% | — | 0.9828 | — | 0.9585 | 0.0029 |
| After | 0.70 | 265,142 | 99.94% | 0.9916 | 0.9833 | +0.0005 | 0.9594 | 0.0028 |
| After | 0.80 | 264,979 | 99.88% | 0.9919 | 0.9823 | −0.0005 | 0.9572 | 0.0030 |
| After | 0.90 | 264,639 | 99.75% | 0.9925 | 0.9824 | −0.0004 | 0.9558 | 0.0026 |
| After | 0.95 | 264,066 | 99.53% | 0.9922 | 0.9801 | −0.0027 | 0.9526 | 0.0035 |
| After | 0.99 | 261,064 | 98.40% | 0.9941 | 0.9813 | −0.0015 | 0.9525 | 0.0027 |

Precision broadly increases with the threshold, from 0.9916 to 0.9941, though not strictly (0.95 is
slightly below 0.90). Coverage barely moves: even at 0.99 the model adopts 98.4% of the pool, so the
threshold never approaches the "few, clean labels" regime it is meant to control. Without the cap,
only the loosest threshold edges above the baseline (+0.0005, well inside the seed-to-seed noise of
Table 1), and the other four fall below it, by up to −0.0027 at 0.95. Attack recall drops at every
threshold from 0.80 upward, by up to 0.006.

## 5. Discussion

**Macro-F1 improves only where labels are scarcest.** At 1%, pseudo-labelling gains +0.0027 and
co-training +0.0031 over the lower baseline. At 5% both methods are marginally below it (−0.0003,
−0.0002), and at 10% the gains (+0.0007, +0.0006) are about one standard deviation in size. The
lower baseline at 5% is already 0.9941 against a same-model ceiling of 0.9961, leaving only 0.0019 of
headroom for any method to compete for. Scarcity, not the algorithm, sets the size of the prize.

**The operational gain is fewer false alarms, and it holds at every budget.** FAR fell for both
methods at all three budgets, including those where macro-F1 did not improve. At 1% it fell from
0.0038 to 0.0023 (pseudo-labelling, −39%) and 0.0020 (co-training, −48%), while attack recall stayed
flat (0.9569 → 0.9581 / 0.9572). On the 75,868 benign test flows that is about 288 → 176 → 148 false
alerts. Co-training cut FAR by 53% at 5% (127 → 60 alerts) and 57% at 10% (143 → 62), reaching 0.0008,
below the full-label ceiling's 0.0021; at those budgets it paid for this with lower recall (0.9840 vs
0.9896 at 5%, 0.9872 vs 0.9911 at 10%). The unlabelled pool, 84.9% benign, sharpened the models'
notion of *normal* traffic more than their notion of *attack*.

**The variance is as large as the effect.** At 1% the per-seed macro-F1 values are:

| Seed | Lower baseline | Pseudo-labelling | Co-training |
|---|---|---|---|
| 42 | 0.9822 | 0.9801 | 0.9796 |
| 43 | 0.9798 | 0.9844 | 0.9849 |
| 44 | 0.9805 | 0.9862 | 0.9873 |

On seed 42 both methods *lose*; on seeds 43 and 44 both win. The standard deviation of the SSL
results (0.0032–0.0039) is comparable to the mean gain (+0.0027/+0.0031), so three seeds provide
evidence of a positive tendency, not a demonstrated improvement. Evaluated on seed 42 alone, this
experiment would have concluded that SSL hurts; the brief's instruction to repeat runs is what
prevents that error.

**The cap, not the threshold, appears to be the effective control.** In the main runs the threshold
excluded almost nothing (94–99% of the pool passed 0.95), so the cap alone determined which rows were
adopted: the top ~1% most confident, at essentially perfect precision. Table 2 shows the opposite
regime: without the cap, adoption rises to 98–100% of the pool at every threshold, roughly 1,550–2,200
adopted labels are wrong, and four of five settings fall below the baseline. Those wrong labels are not
random noise but the model's own errors on the rows it finds hardest, the confirmation-bias failure
mode self-training is known for, and the drop in recall is consistent with an 84.9%-benign pool
importing its imbalance along with its volume. This strongly suggests that the conservative cap is
what allowed pseudo-labelling to help at 1%. It is not a fully controlled result, however: Table 2
has no capped row on the validation set and uses a single seed, so the comparison with Table 1 spans
two different evaluation sets.

**Pseudo-labelling and co-training are statistically indistinguishable.** At 1% they differ by 0.0003
in macro-F1, a tenth of their standard deviation. The diagnostics explain why co-training did not
behave differently: its disagreement-rejection rule **never triggered** (0 conflicts in all 18
rounds). At 1% a third to two-thirds of each learner's selected rows were also selected by the other
learner, and the two agreed on every one of them. After co-training, each 34-feature view on its own
reached 0.9825 and 0.9807 validation macro-F1 at 1%, so both views were individually sufficient, which
is the first condition of Blum & Mitchell (1998). The second condition, that the views make
independent errors, is not met by a random feature split, and the absence of any disagreement
suggests the two learners largely made the same confident predictions. In effect, co-training here
behaved like two capped self-trainers adopting somewhat more unique rows (7,797 vs 5,360 at 1%).

**Sensitivity to the software stack.** The same code, data and seeds were also run locally under
scikit-learn 1.9.1. Within one software stack, two end-to-end runs agreed to all four decimals, but
moving to scikit-learn 1.6.1 on Colab shifted macro-F1 by up to 0.0022 (pseudo-labelling at 1%), the
same order as the SSL effect itself. It also flipped marginal conclusions: pseudo-labelling's gain at 5%
changed sign (+0.0002 → −0.0003), as did the 0.70 row of Table 2 (−0.0006 → +0.0005). The results reported here are those of the
stack that produced the submitted notebook, and the library versions are part of the result.

**Threats to validity.** One dataset, one binary task, one base learner, two SSL rounds, and a
single-seed ablation. CICIDS2017 flow features are strong enough that 2,680 labels already reach 0.98
macro-F1, compressing every effect into the third decimal; a harder dataset would likely show larger
differences. The co-training views were formed at random rather than from semantically independent
feature groups.

## 6. Reproducibility

Seed 42, with 43 and 44 for repeats. Environment of the reported run: Google Colab, Python 3.13.15,
numpy 2.1.3, pandas 2.2.3, scikit-learn 1.6.1, 4 numerical-library threads. Data:
`data/lab1_splits.npz`, exported from Lab 1's `splits.joblib`, SHA-256
`1891044e6bb39ea93549cd7b80f70cd83801efbbe59c69280a0ed90230d8f5ac`. Lab 1's `metrics.py` must be at
`src/metrics.py`. Run `Lab_2_SSL_CICIDS_FIXED.ipynb` top to bottom from a fresh runtime. Section 14
asserts the data source, dimensions, backend, seed set, method coverage, metric validity, split
disjointness and that the ablation cap was disabled; it then writes `lab2_outputs/run_manifest.json`.
Model training for Table 1 took 104 seconds, and the ablation 40 seconds. Because results differ in
the fourth decimal across scikit-learn versions (Section 5), reproducing these exact values requires
the versions above.

## 7. Contribution statement

*To be completed before submission:* `[Name]` …; `[Name]` … .

## References

Blum, A., & Mitchell, T. (1998). Combining labeled and unlabeled data with co-training.
*Proceedings of COLT '98*, 92–100.

Lee, D.-H. (2013). Pseudo-Label: The simple and efficient semi-supervised learning method for deep
neural networks. *ICML Workshop on Challenges in Representation Learning.*

Sharafaldin, I., Lashkari, A. H., & Ghorbani, A. A. (2018). Toward generating a new intrusion
detection dataset and intrusion traffic characterization. *ICISSP 2018.*

Van Engelen, J. E., & Hoos, H. H. (2020). A survey on semi-supervised learning.
*Machine Learning, 109*(2), 373–440.

scikit-learn developers. Semi-supervised learning; `HistGradientBoostingClassifier`.
https://scikit-learn.org/stable/
