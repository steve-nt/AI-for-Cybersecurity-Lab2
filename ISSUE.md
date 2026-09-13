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

Last checked: 2026-09-12, against `Lab_2_SSL_CICIDS_FIXED.ipynb`, which has **never been run**
(15 code cells, 0 executed, no stored outputs). The first check was 2026-09-11 against commit `ad77757`
and `Lab_2_SSL_CICIDS.ipynb`.

---

## Blocking

### [x] I-01 · The notebook uses its own split, not Lab 1's

**Status 2026-09-12: FIXED in code, not yet proven by a run.** The fixed notebook accepts only `data/lab1_splits.npz` (§2 `DATA_MODE = "npz"`), and §3 refuses to continue unless the shapes are 267,984 / 89,328 / 89,329 × 68, the test class counts are 75,868 benign and 13,461 attack, and the attack rate is within 0.01 of 0.1507. It also prints a SHA-256 of the split. The raw-CSV and synthetic paths are gone. The export and the run still have to happen: see I-20.

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

### [x] I-20 · The fixed notebook has never been run, so there are still no results

**Status 2026-09-13: FIXED on Google Colab.** `AI-for-Cybersecurity-Lab2-01/Lab_2_SSL_CICIDS_FIXED.ipynb` executed all 16 code cells with no errors (Python 3.13.15, scikit-learn 1.6.1). Section 3 reproduced digest `1891044e…`, section 5 loaded `src/metrics.py`, and section 14 printed ALL AUTOMATED CHECKS PASSED. Training took 104 s plus 40 s for the ablation, against ~90 min on the local machine, so the local runs were throttled. Note: numbers differ from the local scikit-learn 1.9.1 runs by up to 0.0022 macro-F1; both reports now use the Colab values. Also note `.gitignore` line 30 (`lab2_outputs/`) keeps the result CSVs and Figure 1 out of git.

**Status 2026-09-12 (later): unblocked, still needs the run.** `data/lab1_splits.npz` now exists (40.4 MB, exported from `collab/collab/data/processed/splits.joblib`). Every check in §3 was reproduced against it and passed: shapes 267,984 / 89,328 / 89,329 × 68, test class counts 75,868 benign and 13,461 attack, overall attack rate 0.1507, no NaN or infinity after the float32 cast, 68 unique feature names, and a clean round-trip with `allow_pickle=False`. A `.venv` with numpy, pandas, scikit-learn and joblib is in place (see I-13). What remains is Run All plus committing `lab2_outputs/`.

**Status 2026-09-12 (evening): main results done, notebook still incomplete.** Two full runs of sections 1–10 finished and agree to four decimals; the notebook now stores outputs for 11 of 15 code cells, and `lab2_outputs/` holds the results tables plus the ablation CSV. Both runs were killed by the memory watchdog in the ablation cell, so cells 23–29 (ablation, Figure 1, diagnostics, §14 checks) still have no stored output and §14 has never printed its all-checks-passed line. Table 2 exists because each threshold was run in its own process. To finish: free memory (page cache was holding ~6.3 GB, leaving under 1 GB free) and re-run.
**Where:** `Lab_2_SSL_CICIDS_FIXED.ipynb` (15 code cells, 0 executed) · **Tasks:** T02, T05–T12, T15

**Problem:** The rewrite fixes the code, but nothing has been produced from it. `data/lab1_splits.npz`
doesn't exist, so §3 would stop with `FileNotFoundError` right now, and there's no `lab2_outputs/`
folder. Every number the report needs is still missing, and none of the fixes below are confirmed
until the checks in §14 actually pass on a real run.

**Fix:** run the export snippet in §3 from the Lab 1 environment (it reads
`collab/collab/data/processed/splits.joblib`), then Run All from a fresh kernel. Expect a long CPU
run: three seeds × three budgets × three methods, plus three full-label fits, all on
`HistGradientBoostingClassifier` limited to 4 threads. Then commit `lab2_outputs/`, which is not
ignored.

---

## High

### [x] I-02 · The confidence-threshold ablation doesn't test the threshold

**Status 2026-09-12: FIXED.** §11 passes `apply_class_cap=False` for every ablation run and sweeps 0.70 / 0.80 / 0.90 / 0.95 / 0.99. §14 asserts `passed_threshold == pseudo_labels` to prove the cap really was off, and warns if every threshold still picks the same count. Table 2 now shows `passed_threshold` and pseudo-label precision, so the coverage-versus-quality trade-off is visible.

**Status 2026-09-12 (later): confirmed by running it.** All five thresholds executed: adopted counts are 265,244 / 265,007 / 264,828 / 263,726 / 260,633 out of a 265,304-row pool, i.e. five distinct values with `passed == adopted` in every row, so the cap really is off and the threshold really is the variable. Result: precision rises with the threshold (0.9910 → 0.9938) but no threshold beats the baseline (−0.0003 to −0.0029 macro-F1), because even 0.99 still adopts 98.2% of the pool. Written up as Table 2 in both reports.

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

### [x] I-03 · The upper line isn't Lab 1's score, and Figure 1's caption says it is

**Status 2026-09-12: FIXED.** Lab 1's Random Forest row is hard-coded in §2 `LAB1_REFERENCE` (0.9984 / 0.9968 / 0.9923 / 0.9999 / 0.0006, sourced to the Lab 1 report) and added to Table 1 as its own row. Figure 1 draws it as a green dashed line, and the retrained model is a separate purple dotted "same-model ceiling" that no longer claims to be Lab 1.

**Where:** §2 `LAB1_FULL_METRICS = None`, §8, and the §12 caption · **Tasks:** T05, T12

**Problem:** §8 retrains XGBoost on the notebook's own split (macro-F1 0.9986, FAR 0.0011). The Figure 1
caption calls that line *"the full-label Lab 1 result"*, but Lab 1's number is Random Forest at
0.9968 / 0.0006.

**Fix:** after I-01, keep the retrained upper line: it uses the same model as the SSL methods, so the
comparison is fair. Reword the caption to "full-label upper line (same model, 100% of labels)", and
quote Lab 1's Random Forest score next to it in the report text.

### [x] I-04 · Only one seed was run

**Status 2026-09-12: FIXED.** `SEEDS = (42, 43, 44)`, and §14 refuses to pass unless that is exactly the value. `macro_f1_gain` is now measured against the lower baseline of the *same seed and budget* before averaging, which is more correct than the previous version.

**Where:** §2 `SEEDS = (42,)` · **Tasks:** T04–T12

**Problem:** The brief says that at 1% the exact rows matter, so each setting should be run a few times
and averaged. With one seed, `macro_f1_std` is 0 everywhere and the error bands in Figure 1 are
empty. The observed SSL effects (±0.001 macro-F1) could be smaller than the seed-to-seed noise, and
there's no way to tell.

**Fix:** `SEEDS = (42, 43, 44)`. The whole seed-42 run needed about 90 s of model training on the GPU,
so three seeds is cheap.

### [x] I-05 · The two of you can get different numbers from the same notebook

**Status 2026-09-12: FIXED.** `MODEL_BACKEND = "sklearn"` with no CUDA probe and no fallback; §4 raises if it is changed and §14 checks it again. Thread limits are pinned to 4 instead of following the machine's core count.

**Where:** §2 `MODEL_BACKEND = "auto"`, §4 · **Task:** T15

**Problem:** On a machine without a CUDA build of XGBoost, `auto` silently switches to
`HistGradientBoostingClassifier`, a different model with different results. The run shown used
XGBoost 3.2.0 with CUDA, and this machine has no XGBoost at all.

**Fix:** agree on one value and hard-code it. Use `"sklearn"` if you want it to run anywhere with the
same numbers, or `"xgboost_cuda"` if you both have NVIDIA GPUs (it raises an error instead of silently
switching). Name the model in the report.

### [x] I-06 · No Lab 1 code is reused

**Status 2026-09-12: FIXED, with a packaging catch.** §5 loads `false_alarm_rate` from Lab 1's `metrics.py` and cross-checks it against its own confusion-matrix FAR, failing if they disagree. `REQUIRE_LAB1_METRICS_REUSE = True` makes it mandatory in §14, and the metric names now match Lab 1's (`recall_attack`, `FAR`). The catch: the file it imports lives in the gitignored `collab/`. See I-21.

**Where:** §3 cleaning, §5 `evaluate_model` · **Grading:** "reuses your Lab 1 code" (30% criterion)

**Problem:** The notebook re-implements cleaning and metrics instead of using
`collab/collab/src/clean.py` and `metrics.py`. The metrics are correct, but reuse is marked explicitly.
The column names also differ from Lab 1's CSV (`recall` / `far` vs. `recall_attack` / `FAR`), which
matters if you put the two tables side by side.

**Fix:** have `evaluate_model` call Lab 1's `evaluate_binary` / `false_alarm_rate`
(`sys.path.insert(0, "collab/collab/src")`). After I-01 the data itself comes from Lab 1's cleaning,
so say that in the report as reuse too.

### [ ] I-21 · The notebook demands a file that git doesn't ship
**Where:** §2 `LAB1_METRICS_CANDIDATES`, `REQUIRE_LAB1_METRICS_REUSE = True` · **Tasks:** T15, T16

**Problem:** The Lab 1 metrics reuse from I-06 is mandatory, and §14 fails without it. The first
candidate path is `collab/collab/src/metrics.py`, but `collab/` is gitignored, so it isn't in the repo
or in a zip built from it. Anyone cloning the project, including the marker, gets
`FileNotFoundError: Lab 1 metrics.py was not found`.

**Fix:** copy Lab 1's `metrics.py` to `src/metrics.py` and commit it. That path is the second entry in
`LAB1_METRICS_CANDIDATES` already, and `git check-ignore` confirms `src/` is not ignored. Say in the
report that the file is a copy of Lab 1's, so it still counts as reuse.

---

## Medium

### [ ] I-07 · The per-class cap, not the 0.95 cut-off, controls pseudo-labelling

**Status 2026-09-12: PARTLY ADDRESSED, decision still needed.** The cap stays on in the main runs by design, but it is now measured: every round logs `passed_threshold` next to `accepted_after_cap`, so the report can show when the cap rather than the cut-off did the work. The limit itself is unchanged: with `PSEUDO_TO_TRUE_RATIO_PER_ROUND = 1.0` and 2 rounds, the 1% budget can add at most 5,360 rows, about 2% of the unlabelled pool. Decide before the final run whether to raise the ratio, and say either way in the report.

**Status 2026-09-12 (evening): answered by the ablation — keep the cap.** My earlier suggestion to consider raising the ratio is contradicted by the data. Table 2 removed the cap at the 1% budget: adoption rose to 98–100% of the pool at every threshold and macro-F1 fell below the baseline in all five settings (−0.0003 to −0.0029), because ~1% of 265,000 adopted rows is about 2,400 of the model's own systematic errors. The conservative cap is what produced the +0.0018 and +0.0038 gains, so it should stay as it is and be reported as a deliberate design choice rather than a limitation.

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

### [x] I-08 · The notebook's own cleaning differs from Lab 1's

**Status 2026-09-12: FIXED by deletion.** The `raw_csv`, `presplit_csv` and synthetic modes are gone, together with `DROP_COLUMNS` and the CSV reservoir loader, so the notebook can no longer do its own cleaning. The two remaining mentions of "synthetic" are prose saying it is not accepted.

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

### [x] I-09 · `.gitignore` doesn't protect the notebook's data folder

**Status 2026-09-12: FIXED.** `.gitignore` now has a bare `data/` rule and `git check-ignore` confirms the raw CSVs there are ignored. Good timing: all 8 CSVs, about 880 MB, are now sitting in `data/`. `data/dummy.txt` stays tracked because it was committed before the rule existed, which is harmless.

**Where:** `.gitignore` · **Task:** T16

**Problem:** `raw_csv` mode reads CSVs from `data/`, but `data/*.csv` is not ignored (checked with
`git check-ignore`). A `git add .` with the 8 raw CSVs in place would try to commit ~880 MB. Two related
problems: `data/dummy.txt` is only a placeholder, and the notebook's small outputs (`lab2_outputs/`:
results CSVs, Figure 1, `run_manifest.json`) aren't in the repo.

**Fix:** add `data/*` and `!data/.gitkeep` to `.gitignore`, and swap `dummy.txt` for `.gitkeep`. After
the final run, commit `lab2_outputs/`; nothing in it is ignored and it's all small.

### [x] I-10 · The SSL runs don't record how accurate the guessed labels were

**Status 2026-09-12: FIXED.** Pseudo-labelling logs precision overall and per class each round. Co-training logs `precision_a_to_b`, `precision_b_to_a`, union precision, and each view's stand-alone validation score (§13, saved to `ssl_diagnostics.csv` and `co_training_view_validation.csv`). The true labels are read only after selection, so they cannot influence training. Table 1 gains a `pseudo_label_precision` column weighted by how many rows each round accepted.

**Where:** `history` in §6 and §7 · **Tasks:** T07, T08

**Problem:** Nothing records how many pseudo-labels were correct, even though the hidden true labels
are right there in `y_train`. For co-training, neither view's solo score nor the accuracy of the
labels passed between the learners is logged. Without these, the discussion can't explain *why* SSL
didn't help.

**Fix:** after selection, log `(y[selected] == pseudo_targets).mean()` in `history`. This only
measures; it never feeds a model. For co-training, also score each view's model alone on validation.

### [x] I-11 · The ablation runs at 5% only

**Status 2026-09-12: FIXED.** `ABLATION_BUDGET = 0.01`.

**Where:** §11 `ablation_budget = 0.05` · **Task:** T09

**Problem:** The threshold matters most when labels are scarcest, which is the 1% budget.

**Fix:** run the ablation at 1%, and at 10% if you want to show that the effect shrinks as labels
increase.

### [ ] I-12 · No README or requirements file

**Status 2026-09-12: STILL OPEN.** §15 is better: it lists `threadpoolctl` and `joblib` and says not to submit unless §14 passes. But it is still inside the notebook. `README.md` is still the 28-byte title line and there is no `requirements.txt`. Add the versions from the final run.

**Where:** repo root · **Tasks:** T01, T13 · **Grading:** "a README explains how to run it"

**Problem:** The run instructions live inside §14, and `README.md` is a single title line. There's no
`requirements.txt`. The notebook imports `threadpoolctl` (installed with scikit-learn) and optionally
`xgboost`.

**Fix:** move §14's "How to run" into `README.md`, add the Lab 1 export step from I-01, state the
seed, and add a `requirements.txt` pinned to the versions that ran: Python 3.11.9, scikit-learn 1.9.0,
pandas 3.0.5, XGBoost 3.2.0.

### [ ] I-13 · This machine can't run the notebook yet

**Status 2026-09-12: STILL OPEN.** This machine still has no pandas and no `.venv`. The fixed notebook also needs `threadpoolctl`.

**Status 2026-09-12 (later): mostly done.** `.venv` now has numpy 2.5.3, pandas 3.0.5, scikit-learn 1.9.1, joblib 1.6.0 and threadpoolctl 3.6.0, which is close to the versions the teammate ran (pandas 3.0.5, scikit-learn 1.9.0). Still missing for a full notebook run: `matplotlib` and `jupyter`.

**Where:** local environment · **Task:** T00

**Problem:** System `python3` is 3.13.7 with no pandas or scikit-learn, and there's no `.venv`.

**Fix:** `python3 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt`
(after I-12), plus `jupyter`.

### [ ] I-22 · Two notebooks in the repo, and the invalid one is still there
**Where:** `Lab_2_SSL_CICIDS.ipynb` and `Lab_2_SSL_CICIDS_FIXED.ipynb` · **Task:** T16

**Problem:** The old notebook still holds the stored results from the wrong split. Whoever opens the
repo has to guess which one counts, and the old outputs could end up in the report by mistake.

**Fix:** once I-20 has produced real results, delete the old notebook (its history stays in git), or
rename the new one to `Lab_2_SSL_CICIDS.ipynb` and say in the README which file is the submission.

---

## Low

### [ ] I-14 · The co-training paper is missing from the references

**Status 2026-09-12: STILL OPEN.** The §15 reference list has Lee, Van Engelen & Hoos, scikit-learn and CICIDS2017, but still no Blum & Mitchell, even though co-training is one of the two methods.

**Where:** §14 References · **Task:** T14

**Fix:** add Blum, A. & Mitchell, T. (1998), *Combining Labeled and Unlabeled Data with Co-Training*,
COLT '98.

### [ ] I-15 · Placeholder contribution statement

**Status 2026-09-12: STILL OPEN.** §15 still reads `[Name] implemented and ran ...`.

**Where:** §14 · **Task:** T14 · **Grading:** "who-did-what line included"

**Fix:** replace `[Name] implemented and ran the experiments; [Name] analysed results…` with the real
split of work.

### [ ] I-16 · Figure 1 can't show the differences it's about

**Status 2026-09-12: STILL OPEN.** The y-axis still pads 0.08 below the lowest point (`ylim=(min-0.08, min(1.01, max+0.02))`), so it spans at least 0.10 while the methods differ by far less. There is still no FAR figure, and the caption still does not say the axis is zoomed. The improvement is that both reference lines are drawn and clearly labelled.

**Where:** §12 `ylim` · **Task:** T12

**Problem:** The Y axis runs from about 0.92 to 1.01. All methods fall within 0.0028 of each other,
about 3% of the axis height, so the lines sit almost on top of each other. There's also no FAR figure,
although the brief says to judge on FAR.

**Fix:** zoom the axis to the data (e.g. 0.994–1.000) and say so in the caption. Add a FAR-vs-budget
figure, where the methods actually differ.

### [x] I-17 · The title mentions CICIDS2018, which isn't used

**Status 2026-09-12: FIXED.** The title says CICIDS2017 only.

**Where:** §0 title

**Fix:** say CICIDS2017 only.

### [x] I-18 · Standard deviation uses `ddof=0`

**Status 2026-09-12: FIXED.** `.std(ddof=1)`, and the Table 1 caption says "sample standard deviation".

**Where:** §10 `.std(ddof=0)`

**Problem:** This is the population standard deviation, which reads a little low with only 3 seeds.

**Fix:** use `ddof=1` (sample standard deviation), or state `ddof=0` in the Table 1 caption.

### [ ] I-19 · `TASKS.md` plans a layout the team didn't use

**Status 2026-09-12: STILL OPEN, and now more pressing.** The repo holds two notebooks (see I-22) while `TASKS.md` section 4 still describes the `lab2/src/*.py` layout nobody used.

**Where:** `TASKS.md` section 4 and T01–T04

**Problem:** The plan describes `lab2/src/*.py` scripts with one JSON file per run. The team used one
notebook with CSV output. The `lab2/data/*` rules in `.gitignore` also point at the unused layout.

**Fix:** decide together: either keep the notebook as the single deliverable and trim section 4 of
`TASKS.md` to match, or split the notebook into scripts later.

### [ ] I-23 · The split digest check is available but switched off

**Status 2026-09-12 (later): the value is ready to paste.** Computed with the notebook's own `_split_fingerprint` on the exported file, so it should match what §3 prints:

```python
EXPECTED_LAB1_SHA256 = "1891044e6bb39ea93549cd7b80f70cd83801efbbe59c69280a0ed90230d8f5ac"
```

Set it in §2 after the first run confirms the same digest, and have the other person check that their export produces it too.
**Where:** §2 `EXPECTED_LAB1_SHA256 = None` · **Task:** T15

**Problem:** §3 computes a SHA-256 over the split and would compare it, but with `None` it compares
against nothing. This is exactly the check that proves both of you ran on identical data.

**Fix:** after the first successful run, paste the printed digest into `EXPECTED_LAB1_SHA256` and
have the other person confirm they get the same one.

---

## Findings for the report (not bugs)

From the seed-42 run on the notebook's own split. Re-check all of these after I-01 and I-04.

- Neither SSL method beat the few-label lower line on macro-F1. Pseudo-labelling: −0.0003 / −0.0000 /
  +0.0001 at 1 / 5 / 10%. Co-training: −0.0010 / −0.0005 / −0.0002.
- Both lowered FAR at every budget. Co-training had the lowest FAR (0.0009 / 0.0006 / 0.0007), but it
  also had the lowest attack recall (0.9903 / 0.9955 / 0.9969), so it missed more attacks.
- The lower line was already within 0.002 macro-F1 of the full-label line even at 1%, so SSL had almost
  no room to help. That's the "ceiling problem" `TASKS.md` predicted.
