# Issues and fixes

Small problems found while checking `Lab_2_SSL_CICIDS.ipynb` against the lab brief, Lab 1
(`collab/collab/`) and `TASKS.md`. `TASKS.md` tracks *which task* is done; this file tracks
*specific things to fix*, many of which cut across tasks.

**How to use it:** pick an issue, fix it, tick its box, and add the commit hash after the title,
e.g. `(fixed in a1b2c3d)`. Add new issues at the bottom of the right severity group with the next free
number. Don't renumber old ones.

**Severity:** **Blocking** = results can't be reported until fixed · **High** = costs marks or
breaks reproducibility · **Medium** = weakens the analysis or a deliverable · **Low** = polish.

"§" numbers refer to the notebook's numbered headings (`## 6. Required method…` = §6).

Last checked: 2026-09-11, against commit `ad77757`.

---

## Blocking

### [ ] I-01 · The notebook uses its own split, not Lab 1's
**Where:** §2 `DATA_MODE = "auto"`, resolved to `raw_csv` (see the §3 output) · **Task:** T02

**Problem:** No `data/lab1_splits.npz` existed, so the notebook loaded the 8 raw CSVs and made a new
60/20/20 split. Its own output warns: *"raw_csv mode creates a new split"*. The brief requires the
same cleaning, split and test set as Lab 1.

| | Lab 1 | Notebook run |
|---|---|---|
| Rows | 446,641 (20% sample, duplicates removed) | 2,830,743 (all rows, duplicates kept) |
| Train / val / test | 267,984 / 89,328 / 89,329 | 1,698,445 / 566,149 / 566,149 |
| Features | 68 | 70 |
| Attack rate | 15.07% | 19.70% |

**Fix:** no code change. Export Lab 1's `collab/collab/data/processed/splits.joblib` to
`data/lab1_splits.npz` with keys `X_train, y_train, X_val, y_val, X_test, y_test`:
- `X_*` = Lab 1's raw `X_*`, as numpy arrays;
- `y_*` = Lab 1's `yb_*`;
- `feature_names` as a plain string array, because the notebook loads with `allow_pickle=False`.

Then Run All and check that §3 prints `Resolved data mode: npz` and `train 267984`. Every number
in §8–§12 must be regenerated afterwards.

---

## High

### [ ] I-02 · The confidence-threshold ablation doesn't test the threshold
**Where:** §11, and `choose_confident` in §5 · **Task:** T09

**Problem:** Cut-offs 0.90, 0.95 and 0.99 all added exactly **169,844** pseudo-labels, which is the
per-class cap (42,461 per class per round × 2 classes × 2 rounds). The cap keeps the most confident
rows first, so every cut-off selected the same rows. The three "different" runs were practically the
same experiment (macro-F1 change 0.0000 / 0.0000 / −0.0001), and the brief's required ablation shows
nothing.

**Fix:** in §11 only, lift the cap (`PSEUDO_TO_TRUE_RATIO_PER_ROUND`,
`MAX_PSEUDO_PER_CLASS_PER_ROUND`) so the cut-off decides which rows get in, and say so in the report.
Add lower cut-offs such as 0.70 and 0.80 so the trade-off shows. For each cut-off, log how many rows
passed it and how many of those were correct (see I-10).

### [ ] I-03 · The upper line isn't Lab 1's score, and Figure 1's caption says it is
**Where:** §2 `LAB1_FULL_METRICS = None`, §8, and the §12 caption · **Tasks:** T05, T12

**Problem:** §8 retrains XGBoost on the notebook's own split (macro-F1 0.9986, FAR 0.0011). The Figure 1
caption calls that line *"the full-label Lab 1 result"*, but Lab 1's number is Random Forest at
0.9968 / 0.0006.

**Fix:** after I-01, keep the retrained upper line: it uses the same model as the SSL methods, so the
comparison is fair. Reword the caption to "full-label upper line (same model, 100% of labels)", and
quote Lab 1's Random Forest score next to it in the report text.

### [ ] I-04 · Only one seed was run
**Where:** §2 `SEEDS = (42,)` · **Tasks:** T04–T12

**Problem:** The brief says that at 1% the exact rows matter, so each setting should be run a few times
and averaged. With one seed, `macro_f1_std` is 0 everywhere and the error bands in Figure 1 are
empty. The observed SSL effects (±0.001 macro-F1) could be smaller than the seed-to-seed noise, and
there's no way to tell.

**Fix:** `SEEDS = (42, 43, 44)`. The whole seed-42 run needed about 90 s of model training on the GPU,
so three seeds is cheap.

### [ ] I-05 · The two of you can get different numbers from the same notebook
**Where:** §2 `MODEL_BACKEND = "auto"`, §4 · **Task:** T15

**Problem:** On a machine without a CUDA build of XGBoost, `auto` silently switches to
`HistGradientBoostingClassifier`, a different model with different results. The run shown used
XGBoost 3.2.0 with CUDA, and this machine has no XGBoost at all.

**Fix:** agree on one value and hard-code it. Use `"sklearn"` if you want it to run anywhere with the
same numbers, or `"xgboost_cuda"` if you both have NVIDIA GPUs (it raises an error instead of silently
switching). Name the model in the report.

### [ ] I-06 · No Lab 1 code is reused
**Where:** §3 cleaning, §5 `evaluate_model` · **Grading:** "reuses your Lab 1 code" (30% criterion)

**Problem:** The notebook re-implements cleaning and metrics instead of using
`collab/collab/src/clean.py` and `metrics.py`. The metrics are correct, but reuse is marked explicitly.
The column names also differ from Lab 1's CSV (`recall` / `far` vs. `recall_attack` / `FAR`), which
matters if you put the two tables side by side.

**Fix:** have `evaluate_model` call Lab 1's `evaluate_binary` / `false_alarm_rate`
(`sys.path.insert(0, "collab/collab/src")`). After I-01 the data itself comes from Lab 1's cleaning,
so say that in the report as reuse too.

---

## Medium

### [ ] I-07 · The per-class cap, not the 0.95 cut-off, controls pseudo-labelling
**Where:** `per_class_cap` in §6 and §7 · **Tasks:** T07, T14

**Problem:** The cap is `ceil(labelled × PSEUDO_TO_TRUE_RATIO_PER_ROUND / 2)` per class per round.
Every budget hit it for **both** classes in **both** rounds: 33,968 = 4 × 8,492; 169,844 = 4 × 42,461;
339,688 = 4 × 84,922. Two consequences:
- the 0.95 cut-off never limited anything;
- the pseudo-labels are exactly 50% attack, while the data is 19.7% attack (15.1% on Lab 1's split).

After I-01 the cap gets much tighter. At 1% of Lab 1's training set (2,680 labels), pseudo-labelling
can add at most 5,360 rows, about 2% of the ~265,000 unlabelled rows. So SSL would barely use the
unlabelled pool.

**Fix:** not necessarily a bug, but it is a design choice the report must state. Better, log how many
rows passed the cut-off *before* the cap, and consider a larger ratio so the unlabelled pool is
actually used.

### [ ] I-08 · The notebook's own cleaning differs from Lab 1's
**Where:** §2 `DROP_COLUMNS`, §3 `_clean_and_align` · **Task:** T02

**Problem:** This only matters if the `raw_csv` or `presplit_csv` modes are ever used again:
- `DROP_COLUMNS` lacks Lab 1's `Destination Port`, `Source Port`, `Protocol` and `Fwd Header Length.1`,
  which is why there were 70 features. Lab 1 drops ports as a leakage risk.
- Duplicate rows are not removed. Lab 1 removes them because a duplicate can land in both train and
  test and inflate the score.
- Rows with inf/NaN are kept as NaN; Lab 1 dropped them.
- Row sampling is a random cap, not Lab 1's stratified 20% sample.

**Fix:** once I-01 is done, either copy Lab 1's `ID_COLUMNS` and deduplication into §3, or remove the
CSV modes so nobody runs them by accident.

### [ ] I-09 · `.gitignore` doesn't protect the notebook's data folder
**Where:** `.gitignore` · **Task:** T16

**Problem:** `raw_csv` mode reads CSVs from `data/`, but `data/*.csv` is not ignored (checked with
`git check-ignore`). A `git add .` with the 8 raw CSVs in place would try to commit ~880 MB. Two related
problems: `data/dummy.txt` is only a placeholder, and the notebook's small outputs (`lab2_outputs/`:
results CSVs, Figure 1, `run_manifest.json`) aren't in the repo.

**Fix:** add `data/*` and `!data/.gitkeep` to `.gitignore`, and swap `dummy.txt` for `.gitkeep`. After
the final run, commit `lab2_outputs/`; nothing in it is ignored and it's all small.

### [ ] I-10 · The SSL runs don't record how accurate the guessed labels were
**Where:** `history` in §6 and §7 · **Tasks:** T07, T08

**Problem:** Nothing records how many pseudo-labels were correct, even though the hidden true labels
are right there in `y_train`. For co-training, neither view's solo score nor the accuracy of the
labels passed between the learners is logged. Without these, the discussion can't explain *why* SSL
didn't help.

**Fix:** after selection, log `(y[selected] == pseudo_targets).mean()` in `history`. This only
measures; it never feeds a model. For co-training, also score each view's model alone on validation.

### [ ] I-11 · The ablation runs at 5% only
**Where:** §11 `ablation_budget = 0.05` · **Task:** T09

**Problem:** The threshold matters most when labels are scarcest, which is the 1% budget.

**Fix:** run the ablation at 1%, and at 10% if you want to show that the effect shrinks as labels
increase.

### [ ] I-12 · No README or requirements file
**Where:** repo root · **Tasks:** T01, T13 · **Grading:** "a README explains how to run it"

**Problem:** The run instructions live inside §14, and `README.md` is a single title line. There's no
`requirements.txt`. The notebook imports `threadpoolctl` (installed with scikit-learn) and optionally
`xgboost`.

**Fix:** move §14's "How to run" into `README.md`, add the Lab 1 export step from I-01, state the
seed, and add a `requirements.txt` pinned to the versions that ran: Python 3.11.9, scikit-learn 1.9.0,
pandas 3.0.5, XGBoost 3.2.0.

### [ ] I-13 · This machine can't run the notebook yet
**Where:** local environment · **Task:** T00

**Problem:** System `python3` is 3.13.7 with no pandas or scikit-learn, and there's no `.venv`.

**Fix:** `python3 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt`
(after I-12), plus `jupyter`.

---

## Low

### [ ] I-14 · The co-training paper is missing from the references
**Where:** §14 References · **Task:** T14

**Fix:** add Blum, A. & Mitchell, T. (1998), *Combining Labeled and Unlabeled Data with Co-Training*,
COLT '98.

### [ ] I-15 · Placeholder contribution statement
**Where:** §14 · **Task:** T14 · **Grading:** "who-did-what line included"

**Fix:** replace `[Name] implemented and ran the experiments; [Name] analysed results…` with the real
split of work.

### [ ] I-16 · Figure 1 can't show the differences it's about
**Where:** §12 `ylim` · **Task:** T12

**Problem:** The Y axis runs from about 0.92 to 1.01. All methods fall within 0.0028 of each other,
about 3% of the axis height, so the lines sit almost on top of each other. There's also no FAR figure,
although the brief says to judge on FAR.

**Fix:** zoom the axis to the data (e.g. 0.994–1.000) and say so in the caption. Add a FAR-vs-budget
figure, where the methods actually differ.

### [ ] I-17 · The title mentions CICIDS2018, which isn't used
**Where:** §0 title

**Fix:** say CICIDS2017 only.

### [ ] I-18 · Standard deviation uses `ddof=0`
**Where:** §10 `.std(ddof=0)`

**Problem:** This is the population standard deviation, which reads a little low with only 3 seeds.

**Fix:** use `ddof=1` (sample standard deviation), or state `ddof=0` in the Table 1 caption.

### [ ] I-19 · `TASKS.md` plans a layout the team didn't use
**Where:** `TASKS.md` section 4 and T01–T04

**Problem:** The plan describes `lab2/src/*.py` scripts with one JSON file per run. The team used one
notebook with CSV output. The `lab2/data/*` rules in `.gitignore` also point at the unused layout.

**Fix:** decide together: either keep the notebook as the single deliverable and trim section 4 of
`TASKS.md` to match, or split the notebook into scripts later.

---

## Findings for the report (not bugs)

From the seed-42 run on the notebook's own split. Re-check all of these after I-01 and I-04.

- Neither SSL method beat the few-label lower line on macro-F1. Pseudo-labelling: −0.0003 / −0.0000 /
  +0.0001 at 1 / 5 / 10%. Co-training: −0.0010 / −0.0005 / −0.0002.
- Both lowered FAR at every budget. Co-training had the lowest FAR (0.0009 / 0.0006 / 0.0007), but it
  also had the lowest attack recall (0.9903 / 0.9955 / 0.9969), so it missed more attacks.
- The lower line was already within 0.002 macro-F1 of the full-label line even at 1%, so SSL had almost
  no room to help. That's the "ceiling problem" `TASKS.md` predicted.
