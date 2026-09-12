# Learning When Labels Are Scarce: Semi-Supervised Intrusion Detection on CICIDS2017

**Lab 2, D7084E/D7041E.** Companion plain-language version: `Report-Plain-Language.md`.
All results below come from `Lab_2_SSL_CICIDS_FIXED.ipynb`, run on the verified Lab 1 split,
seeds 42/43/44, 2026-09-12.

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
Lab 1's split, and aborts unless the three shapes, the test class counts (75,868 benign / 13,461
attack), and the overall attack rate match Lab 1 exactly. It also records a SHA-256 digest of the
arrays, so two machines can prove they ran on identical data. Lab 1's `false_alarm_rate` is imported
and cross-checked against an independent confusion-matrix computation at every evaluation.

**Label budgets.** From the training set only, a stratified subset of 1%, 5% or 10% retains its
labels (2,680 / 13,399 / 26,798 flows); the remaining 265,304 / 254,585 / 241,186 flows form the
unlabelled pool. Validation and test labels are never hidden and never used for fitting. Each
configuration is repeated with seeds 42, 43 and 44, and results are averaged; at the 1% budget the
identity of the retained rows is itself a source of variance (Section 5).

**Base learner.** One model is used for every method so comparisons are attributable to the method
and not the estimator: scikit-learn `HistGradientBoostingClassifier` (learning rate 0.08, ≤120
iterations, 31 leaves, L2 = 1.0, early stopping on an internal 10% split), with class-balanced
sample weights. The backend is pinned; there is no hardware-dependent fallback.

**Metrics.** Accuracy, macro-F1, attack recall, ROC-AUC and false alarm rate (FAR = FP/(FP+TN)).
Given 84.9% benign traffic, accuracy is reported only for completeness; macro-F1 and FAR carry the
argument, with recall as the operational safety check. The test set is evaluated once per configured
model, after all design decisions are fixed.

## 3. Methods

**Pseudo-labelling (required).** Self-training in the sense of Lee (2013). The model trained on the
labelled slice predicts the unlabelled pool; predictions at or above a confidence threshold of 0.95
are adopted as labels and the model is refitted, for 2 rounds. Adopted rows carry half the weight of
genuine labels, and a per-class cap of `⌈n_labelled/2⌉` per round prevents confident benign
predictions from swamping the scarce attack class. The true labels of adopted rows are read **after**
selection only, to audit precision; that audit never influences selection, fitting or stopping.

**Co-training (second method).** Following Blum & Mitchell (1998), the 68 features are split by a
seeded permutation into two disjoint views of 34. A learner is trained per view; each passes its
confident predictions to the *other* learner, and rows on which the two disagree are rejected.
Final predictions average the two views' probabilities. Co-training was chosen over Mean Teacher and
FixMatch because it needs no neural network and, more importantly, no data augmentation: there is no
established semantics-preserving perturbation for tabular flow records, which makes consistency-based
methods hard to justify here.

## 4. Results

**Table 1.** Test-set performance by label budget. Lab 2 rows are means over seeds 42/43/44, with
the sample standard deviation of macro-F1; Δ is each SSL method's mean per-seed macro-F1 gain over
the same-seed, same-budget lower baseline. The final row is the original Lab 1 Random Forest, not a
retrained substitute.

| Budget | Method | Accuracy | Macro-F1 | Δ vs lower | Recall | ROC-AUC | FAR |
|---|---|---|---|---|---|---|---|
| 1% | Few-label lower baseline | 0.9896 | 0.9795 ± 0.0029 | — | 0.9566 | 0.9954 | 0.0045 |
| 1% | Pseudo-labelling | 0.9906 | 0.9814 ± 0.0029 | **+0.0018** | 0.9572 | 0.9962 | 0.0035 |
| 1% | Co-training | 0.9916 | **0.9833 ± 0.0047** | **+0.0038** | 0.9561 | 0.9974 | **0.0021** |
| 5% | Few-label lower baseline | 0.9969 | 0.9940 ± 0.0001 | — | 0.9894 | 0.9995 | 0.0017 |
| 5% | Pseudo-labelling | 0.9970 | 0.9942 ± 0.0006 | +0.0002 | 0.9870 | 0.9994 | 0.0012 |
| 5% | Co-training | 0.9968 | 0.9938 ± 0.0017 | −0.0003 | 0.9836 | 0.9994 | 0.0008 |
| 10% | Few-label lower baseline | 0.9971 | 0.9943 ± 0.0006 | — | 0.9916 | 0.9997 | 0.0019 |
| 10% | Pseudo-labelling | 0.9973 | 0.9947 ± 0.0007 | +0.0004 | 0.9891 | 0.9997 | 0.0013 |
| 10% | Co-training | 0.9973 | 0.9946 ± 0.0005 | +0.0003 | 0.9874 | 0.9997 | 0.0010 |
| 100% | Full-label same-model ceiling | 0.9981 | 0.9962 ± 0.0003 | — | 0.9982 | 1.0000 | 0.0020 |
| 100% | **Lab 1 full-label Random Forest** | 0.9984 | 0.9968 | — | 0.9923 | 0.9999 | 0.0006 |

**Figure 1** (`lab2_outputs/macro_f1_vs_label_budget.png`) plots mean test macro-F1 against the
labelled fraction, with one-standard-deviation bands, the Lab 1 Random Forest as a dashed reference
line and the same-model 100% ceiling as a dotted line. It shows the pattern of Table 1: a visible
separation between the three curves at 1% that closes by 5%.

**Table 2.** Threshold ablation at the 1% budget, seed 42, scored on **validation** (the test set is
not touched). Only the confidence threshold changes between rows. The per-class cap is disabled for
every "after" row, so adoption is governed by the threshold alone; `passed = adopted` in every row
confirms this. Precision is measured against the hidden labels after selection, for analysis only.
The unlabelled pool contains 265,304 flows.

| Setting | Threshold | Passed = adopted | Share of pool | Pseudo-label precision | Macro-F1 | Δ | Recall | FAR |
|---|---|---|---|---|---|---|---|---|
| Before: few-label baseline | — | 0 | 0% | — | 0.9824 | — | 0.9562 | 0.0027 |
| After | 0.70 | 265,244 | 99.98% | 0.9910 | 0.9818 | −0.0006 | 0.9546 | 0.0028 |
| After | 0.80 | 265,007 | 99.89% | 0.9913 | 0.9812 | −0.0012 | 0.9536 | 0.0030 |
| After | 0.90 | 264,828 | 99.82% | 0.9918 | 0.9821 | −0.0003 | 0.9554 | 0.0028 |
| After | 0.95 | 263,726 | 99.40% | 0.9924 | 0.9817 | −0.0007 | 0.9536 | 0.0027 |
| After | 0.99 | 260,633 | 98.24% | 0.9938 | 0.9795 | −0.0029 | 0.9441 | 0.0022 |

Two things are visible. Precision increases monotonically with the threshold, from 0.9910 to 0.9938,
exactly as the method's rationale predicts. Yet **no threshold improves on the baseline**, and the
strictest is the worst (−0.0029), because the threshold barely restricts coverage: even at 0.99 the
model is confident enough to adopt 98.2% of the pool. The confidence distribution is saturated, so
the threshold cannot deliver the "few, clean labels" regime it is supposed to control.

**Pseudo-label quality.** Adopted labels were essentially always correct: precision 1.0000 for
pseudo-labelling at all three budgets, and 0.9974 / 1.0000 / 0.9999 for co-training. Adoption volumes
were 5,360 / 26,800 / 53,596 rows for pseudo-labelling and 7,802 / 41,615 / 74,635 for co-training.

## 5. Discussion

**SSL helps where labels are scarcest, and nowhere else.** At 1%, co-training gains +0.0038 macro-F1
and pseudo-labelling +0.0018 over the lower baseline; at 5% and 10% every gain falls within
±0.0004, i.e. nothing. The lower baseline at 5% is already 0.9940 against a full-label ceiling of
0.9962, leaving 0.0022 of headroom for any method to compete for. Scarcity, not the algorithm, sets
the size of the prize.

**The operational gain is in false alarms, not detection.** At 1%, FAR falls from 0.0045 to 0.0035
(pseudo-labelling) and 0.0021 (co-training), a reduction of 22% and 53%, while recall is unchanged
(0.9566 → 0.9572 / 0.9561). On the 75,868 benign test flows this is roughly 345 → 162 false alerts.
The unlabelled pool therefore sharpened the model's notion of *normal* rather than its notion of
*attack*, which is what one would expect when the pool is 84.9% benign. Notably, co-training at 5%
and 10% attains a lower FAR (0.0008, 0.0010) than the full-label ceiling (0.0020), though at lower
recall — the methods trade in the direction of caution.

**The variance is as large as the effect.** At 1% the per-seed macro-F1 values are:

| Seed | Lower baseline | Pseudo-labelling | Co-training |
|---|---|---|---|
| 42 | 0.9822 | 0.9786 | 0.9779 |
| 43 | 0.9764 | 0.9844 | 0.9856 |
| 44 | 0.9800 | 0.9811 | 0.9864 |

On seed 42 both methods *lose*; on seeds 43 and 44 both win. The standard deviation (0.0029–0.0047)
is comparable to the mean gain (+0.0018/+0.0038), so with three seeds this is evidence of a positive
tendency, not a demonstrated improvement. An earlier single-seed version of this experiment used
seed 42 alone and concluded the opposite. This is the clearest methodological lesson of the lab, and
it is why the brief's instruction to repeat runs matters.

**The cap, not the threshold, is the effective control — and it is what made SSL work.** Pseudo-label
precision of ~1.0 in Table 1 shows the 0.95 threshold plus the per-class cap admit almost no wrong
labels, at the cost of using only 5,360 of 265,304 available flows (2.0%). Table 2 shows what
happens when that cap is removed: adoption jumps to 98–100% of the pool at every threshold,
precision falls to 0.991–0.994, and macro-F1 drops below the baseline in all five settings. Roughly
1% of 265,000 adopted rows is about 2,400 wrong labels, and they are not random noise — they are the
model's own systematic errors on the rows it finds hardest, which is precisely the confirmation-bias
failure mode the method is warned about. Recall falls in every uncapped setting (0.9562 → 0.9441 at
the strictest threshold) because an 84.9%-benign pool imports its class imbalance along with its
volume, despite class-balanced weights.

The consequence for interpretation is important: the +0.0018 and +0.0038 gains in Table 1 are
attributable to the conservative adoption policy, not to the threshold. Tuning the threshold on this
dataset is close to useless, whereas capping how much is adopted is decisive. A study that reported
only a threshold sweep would have concluded that pseudo-labelling does not work here.

**Why co-training beats pseudo-labelling at 1%.** Co-training adopted 7,802 rows against 5,360, and
its disagreement-rejection rule discards exactly the rows where the two views conflict. Two learners
on disjoint feature views make partially independent errors, so the cross-teaching step corrects
blind spots that a single self-training model would instead reinforce. Its higher variance
(± 0.0047) is consistent with this: a view split that happens to be unbalanced hurts.

**Threats to validity.** One dataset, one binary task, one base learner and two SSL rounds. The
CICIDS2017 flow features are strong enough that even 2,680 labels reach 0.98 macro-F1, compressing
all effects into the third decimal; a harder dataset would likely show larger differences. The 68
features were split into views at random, so the two views are not semantically independent in the
way Blum & Mitchell's formulation assumes.

## 6. Reproducibility

Seed 42 (with 43 and 44 for repeats); `data/lab1_splits.npz` exported from Lab 1's `splits.joblib`
with digest `1891044e6bb39ea93549cd7b80f70cd83801efbbe59c69280a0ed90230d8f5ac`; Python 3.13.7,
numpy 2.5.3, pandas 3.0.5, scikit-learn 1.9.1, matplotlib 3.11.2. Run `Lab_2_SSL_CICIDS_FIXED.ipynb` top to bottom;
section 14 asserts the data source, dimensions, backend, seed set, method coverage, metric validity
and split disjointness, and writes `lab2_outputs/run_manifest.json`. Total model-training time for
the reported results was approximately 90 minutes on 4 CPU threads.

Table 2 was produced by running each threshold in a separate process, executing the notebook's own
setup cells (environment, configuration, loader, backend, metrics, pseudo-labeller) so the code path
is identical; the notebook's ablation cell exceeded the available memory of the machine used here
when run inside the full kernel. The base model is the same deterministic fit the main loop uses for
seed 42 at the 1% budget.

The experiment was executed end to end twice, in separate processes on separate occasions. Every
value in Table 1 — each mean, each standard deviation, each per-seed macro-F1 and each gain —
agreed to within 5 x 10^-5, i.e. to all four reported decimals; only wall-clock timings differed.
Determinism therefore rests on the fixed seeds, the pinned backend and the fixed thread count, not
on chance.

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
