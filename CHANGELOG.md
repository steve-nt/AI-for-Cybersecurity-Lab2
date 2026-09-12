# Changelog

Every task from `TASKS.md` gets one entry here, written **when the task is finished**.
An entry says what was done, why it was done that way, and where the idea came from.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); dates are `YYYY-MM-DD`.

**House rules**
- One entry per task, tagged with its task ID (`T07`) and the person who did it (`@A` / `@B`).
- Say *why*, not just *what* — "threshold 0.95, because 0.80 let 12% wrong pseudo-labels in" beats "added pseudo-labelling".
- Record numbers when a task produces them. This file becomes the skeleton of the report.
- Cite anything you copied or adapted: a doc page, a tutorial, a paper, a Lab 1 file.
- Never rewrite an old entry. If something turned out wrong, add a new entry saying so.

**Copy-paste template**

```markdown
### [Txx] Short title — @A — YYYY-MM-DD
**What:** one or two sentences.
**Why:** the reasoning, especially any choice you had to make.
**Result:** files produced, and the numbers if there are any.
**Sources:** links or citations.
```

---

## [Unreleased]

### [T02] Exported Lab 1's split to `data/lab1_splits.npz` — 2026-09-12
**What:** Converted `collab/collab/data/processed/splits.joblib` (489 MB) into
`data/lab1_splits.npz` (40.4 MB), the file `Lab_2_SSL_CICIDS_FIXED.ipynb` §3 requires. Also created a
`.venv` with numpy 2.5.3, pandas 3.0.5, scikit-learn 1.9.1, joblib 1.6.0 and threadpoolctl 3.6.0,
because the joblib file contains pandas DataFrames and a fitted scikit-learn scaler and cannot be read
without them.

**Why:** A plain copy-and-rename cannot work: `splits.joblib` is a pickled Python dictionary, while
the notebook calls `np.load(..., allow_pickle=False)` and expects the keys `X_train, y_train, X_val,
y_val, X_test, y_test, feature_names`. The export takes Lab 1's **raw** (unscaled) features, since
`HistGradientBoostingClassifier` needs no scaling and Lab 1's Random Forest reference was trained on
raw features too; `yb_*` becomes `y_*` as int8; and `feature_names` is written as a plain Unicode
string array so the file opens without pickling. Features are cast to float32, which is what the
notebook does on load anyway, and it keeps the file small.

**Result:** every check in §3 reproduced and passed.

| Check | Value |
|---|---|
| Shapes | train 267,984 × 68, validation 89,328 × 68, test 89,329 × 68 |
| Test class counts | 75,868 benign / 13,461 attack (matches the notebook's expected values) |
| Attack rate | 0.1507 in all three splits |
| Finite after float32 | yes, no NaN or infinity |
| Feature names | 68, all unique, starting `Flow Duration`, `Total Fwd Packets`, … |
| Round-trip `allow_pickle=False` | identical arrays |
| Seed in the Lab 1 bundle | 42 |

The notebook's own `_split_fingerprint` gives
`1891044e6bb39ea93549cd7b80f70cd83801efbbe59c69280a0ed90230d8f5ac`, ready for `EXPECTED_LAB1_SHA256`
(I-23). The `.npz` is gitignored by the `data/` rule, so each person exports their own copy and the
digest is what proves they match. This clears the blocker in front of I-20; the notebook itself still
has to be run.

**Sources:** `Lab_2_SSL_CICIDS_FIXED.ipynb` §3 (export recipe and validation rules); Lab 1
`collab/collab/src/prepare.py` (which produced the split).

### [Review] Checked the rewritten notebook against ISSUE.md — 2026-09-12
**What:** Read `Lab_2_SSL_CICIDS_FIXED.ipynb` (31 cells, 15 code) and checked each of the 19 issues.
Ticked 12 as fixed, left 7 open, and added 4 new ones (I-20 to I-23) with a dated status line on
every issue.

**Why:** The rewrite is a real improvement: it accepts only the exact Lab 1 split and refuses
anything else, pins the model to scikit-learn so two machines can't diverge, runs three seeds,
disables the pseudo-label cap during the ablation so the threshold is actually the variable under
test, reuses Lab 1's `false_alarm_rate` and cross-checks it, and adds pseudo-label precision
diagnostics. But the notebook has **0 executed cells and no stored outputs**, `data/lab1_splits.npz`
doesn't exist, and there's no `lab2_outputs/`, so the lab still has no results.

**Result:** fixed: I-01 (split, in code), I-02 (ablation cap), I-03 (Lab 1 upper line and caption),
I-04 (three seeds), I-05 (fixed backend), I-06 (Lab 1 metric reuse), I-08 (cleaning modes removed),
I-09 (`data/` now ignored, which matters because the 8 raw CSVs are now in `data/`), I-10
(pseudo-label precision and per-view scores), I-11 (ablation at 1%), I-17 (title), I-18 (`ddof=1`).
Still open: I-07, I-12 to I-16, I-19. New:
- I-20 (blocking): the notebook has never been run, so no results exist;
- I-21: `REQUIRE_LAB1_METRICS_REUSE` points at `collab/collab/src/metrics.py`, which is gitignored, so
  a clean clone fails; copy it to `src/metrics.py`, which is not ignored;
- I-22: both the old and the fixed notebook are in the repo;
- I-23: the split SHA-256 check exists but is set to `None`.

**Sources:** `Lab_2_SSL_CICIDS_FIXED.ipynb` §2–§15; Lab 1 `collab/collab/src/metrics.py`;
`git check-ignore` for I-09 and I-21.

### [Review] Issue list for the Lab 2 notebook — 2026-09-11
**What:** Added `ISSUE.md`, which lists 19 problems found while checking `Lab_2_SSL_CICIDS.ipynb`
against the lab brief, Lab 1 and `TASKS.md`. Each one has a severity, a location, the problem, a fix,
and the task it relates to.

**Why:** The status lines in `TASKS.md` say what each task is still missing. Fixes that span several
tasks (reproducibility, `.gitignore`, captions, references) had no place to live. A separate
checklist lets either person pick one up and tick it off without editing the task descriptions.

**Result:** 1 blocking, 5 high, 7 medium and 6 low issues. The most important besides the wrong split
(I-01):
- the threshold ablation selects the same rows at every cut-off (I-02);
- the notebook can silently switch models on another machine (I-05);
- after the switch to Lab 1's split, the per-class cap would let pseudo-labelling use only about 2% of
  the unlabelled pool at the 1% budget (I-07);
- `.gitignore` doesn't cover the raw CSVs in `data/` (I-09).

**Sources:** Lab 2 brief (steps 2–6 and the grading table); Lab 1 `collab/collab/src/config.py`
(`ID_COLUMNS`), `clean.py` and `metrics.py`; `git check-ignore` for I-09.

### [T01, T03, T04, T06, T07, T08, T11, T12] Whole experiment in one notebook — @kirsil-5 — 2026-09-10
**What:** Added `Lab_2_SSL_CICIDS.ipynb`, 29 cells run end to end on the 8 CICIDS2017 CSVs. It
contains a settings cell, data loading and cleaning, stratified 1 / 5 / 10% label budgets, the
few-label lower line, iterative pseudo-labelling, co-training, a full-label upper line, a
confidence-threshold ablation scored on validation only, a results table (Table 1), a macro-F1 curve
(Figure 1), automated sanity checks and a run manifest.

**Why:** One notebook that runs top to bottom with Run All matches the brief's "notebook or
scripts" option. Choices recorded in it:
- one base model for every method: XGBoost on the GPU, falling back to `HistGradientBoostingClassifier`;
- class-balanced sample weights, with each pseudo-label worth 0.5 of a real label;
- confidence cut-off 0.95, 2 rounds, and a per-class cap on pseudo-labels added per round;
- co-training on a random, seeded 35/35 feature split that rejects rows the two learners disagree on.

**Result** (test set, seed 42 only, macro-F1 / FAR):

| Budget | Few-label lower line | Pseudo-labelling | Co-training |
|---|---|---|---|
| 1% | 0.9969 / 0.0016 | 0.9965 / 0.0012 | 0.9958 / 0.0009 |
| 5% | 0.9983 / 0.0010 | 0.9983 / 0.0008 | 0.9978 / 0.0006 |
| 10% | 0.9984 / 0.0011 | 0.9985 / 0.0009 | 0.9982 / 0.0007 |
| 100% (retrained upper line) | 0.9986 / 0.0011 | | |

Neither SSL method beat the lower line on macro-F1. Both lowered FAR at every budget. Output files
went to `lab2_outputs/` on the author's machine and are not committed.

**Not yet valid for the report** (checked 2026-09-11; details under Progress in `TASKS.md`):
- It ran in `raw_csv` mode and built a new split (1,698,445 / 566,149 / 566,149 rows, 70 features,
  19.70% attack) instead of using Lab 1's (267,984 / 89,328 / 89,329 rows, 68 features, 15.07%
  attack). Duplicates were not removed, and `Destination Port` and the duplicated
  `Fwd Header Length.1` were kept. Fix: re-run on Lab 1's split (T02).
- The threshold ablation added exactly 169,844 rows at every cut-off because the per-class cap
  decided the count, so it does not measure the cut-off yet (T09).
- Only seed 42 was run, and the upper line is retrained rather than Lab 1's score (T05).

**Sources:** Lee (2013), *Pseudo-Label*; Van Engelen & Hoos (2020), *A survey on semi-supervised
learning*; Blum & Mitchell (1998), *Combining Labeled and Unlabeled Data with Co-Training* (missing
from the notebook's own reference list, so add it to the report);
[scikit-learn semi-supervised learning guide](https://scikit-learn.org/stable/modules/semi_supervised.html);
[scikit-learn `HistGradientBoostingClassifier`](https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.HistGradientBoostingClassifier.html);
[XGBoost GPU support](https://xgboost.readthedocs.io/en/stable/gpu/index.html);
[CICIDS2017](https://www.unb.ca/cic/datasets/ids-2017.html).

### [T-init] Project scaffolding: task list, .gitignore, changelog — 2026-09-09
**What:** Read both lab briefs and the finished Lab 1 project in `collab/collab/`, then wrote
`TASKS.md` (17 tasks, T00–T16), this `CHANGELOG.md`, and `.gitignore`.

**Why:** Two people are working in parallel, so the tasks are built around **file contracts**
rather than shared code. The key one is *one JSON file per experiment run* in
`lab2/results/runs/` (task T03): both people can run experiments all day and never edit the same
file, so there are no merge conflicts, and the final results table (T11) is just "read every JSON
in that folder". Only `T01 → T02/T03 → T04` is a genuine bottleneck; after that the two
semi-supervised methods (T07 pseudo-labelling, T08 co-training) are completely independent.

**Result:**
- `TASKS.md` — each task states owner, prerequisites, the exact file it produces, a plain-language
  explanation of the idea, steps, a "done when" checklist and its sources.
- `.gitignore` — excludes `collab/` (1.6 GB of Lab 1 data), the Lab 2 data snapshot, trained
  models, virtualenvs and notebook checkpoints; deliberately **keeps** results tables, figures and
  per-run JSON so the numbers are reviewable in git.
- `CHANGELOG.md` — this file.

**Facts established while reading Lab 1**, which the Lab 2 plan is built on:
- Lab 1 used all 8 CICIDS2017 day files, `SAMPLE_FRACTION = 0.20`, `SEED = 42` → 446,641 rows × 68
  features, 84.9% BENIGN / 15.1% attack, split 60/20/20 into 267,984 / 89,328 / 89,329 rows.
- `collab/collab/data/processed/splits.joblib` already holds that exact split, scaled, with binary
  and multiclass labels — Lab 2 reuses it directly instead of re-splitting, which is what makes the
  two labs comparable.
- Lab 1 upper line on the test set (Random Forest, 300 trees, depth 20, 100% of labels):
  accuracy 0.9984, macro-F1 0.9968, recall 0.9923, ROC-AUC 0.9999, FAR 0.0006.
- `collab/collab/src/metrics.py` already computes all five metrics the Lab 2 brief asks for, so
  Lab 2 imports it unchanged rather than reimplementing it (code reuse is a graded criterion).
- Consequence flagged in `TASKS.md`: with a 0.9968 ceiling and ~2,680 labelled rows even at the 1%
  budget, the few-labels-only baseline will already be strong and the room for SSL to help is
  small. Optional extra 0.1% budget, or repeating with Logistic Regression, makes the effect
  visible. This is a result to report, not a bug to hide.

**Sources:** Lab 2 brief `2. Lab2_Learning_when_labels_are_scarce.pdf`; Lab 1 code in
`collab/collab/`; [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
[GitHub gitignore templates (Python)](https://github.com/github/gitignore/blob/main/Python.gitignore).

---

## Pending

Move each line up into `[Unreleased]` as a full entry when the task is done.

- [ ] `T00` environment set up, Lab 1 `splits.joblib` present on both machines — @B *(partial: teammate's machine only)*
- [x] `T01` `lab2/` skeleton and `src/config.py` (SEED, BUDGETS, base model) — @A *(done in notebook §2)*
- [ ] `T02` frozen data snapshot + fingerprint — @A *(blocking: notebook used a new split)*
- [x] `T03` scoring helper and per-run JSON writer — @B *(done in notebook §5/§10, as CSV)*
- [x] `T04` stratified 1% / 5% / 10% label splits — @B *(done in notebook §5)*
- [ ] `T05` upper line (Lab 1, 100% labels) — @A *(partial: retrained, not Lab 1's score)*
- [x] `T06` lower line (few labels only) — @B *(done in notebook §9)*
- [x] `T07` pseudo-labelling (required method) — @A *(done in notebook §6/§9)*
- [x] `T08` co-training (second method) — @B *(done in notebook §7/§9)*
- [ ] `T09` ablation: confidence cut-off — @A *(partial: cap masks the cut-off)*
- [ ] `T10` ablation: unlabelled pool size (optional) — @B
- [x] `T11` results table — @A *(done in notebook §10)*
- [x] `T12` macro-F1 and FAR curves — @B *(macro-F1 done in notebook §12; FAR figure not made)*
- [ ] `T13` `lab2/README.md` — @B *(partial: "How to run" inside notebook §14)*
- [ ] `T14` report PDF (2–3 pages + appendix) — @A + @B
- [ ] `T15` reproducibility check on both machines — @A + @B
- [ ] `T16` submission to Canvas — @A + @B
