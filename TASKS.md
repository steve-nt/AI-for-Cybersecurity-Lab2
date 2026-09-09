# Lab 2 — Learning When Labels Are Scarce: task list

**Team:** two people. Every task below says who owns it, what it needs before it can start,
and exactly which file it produces. Tasks that do not share a file can be done at the same
time, on different machines, without talking to each other.

**Assumed knowledge: none.** Every task explains the idea in plain words before the steps.

---

## 0. The one-paragraph version of this lab

In Lab 1 you built an intrusion detector: you gave a model 268,000 network connections
where somebody had already written "this one is normal, this one is an attack", and it
learned to tell them apart. In real life nobody hands you that. Labelling traffic means a
human analyst looking at connections one by one, so you typically have labels for a tiny
slice and a mountain of unlabelled traffic.

**Semi-supervised learning (SSL)** is the family of tricks that squeezes value out of the
unlabelled mountain. Lab 2 asks you to *pretend* you only have 1%, 5% and 10% of your Lab 1
labels, hide the rest, and see how close SSL gets you back to the Lab 1 score.

Every number you produce must sit between two reference lines:

| Line | What it is | Where it comes from |
|---|---|---|
| **Upper line** | Lab 1 model trained on **100%** of the labels | Already computed: `collab/collab/results/tables/binary_results.csv` |
| **Lower line** | Same model trained on **only the 1% / 5% / 10% slice** | Task T06 |
| **Your SSL result** | Pseudo-labelling (T07) and co-training (T08) | Must beat the lower line to mean anything |

If the SSL number does not beat the lower line, SSL did not help. Reporting that honestly is
worth more marks than hiding it.

---

## 1. Words you need (read once, 3 minutes)

| Word | Plain meaning |
|---|---|
| **Feature** | One measured number about a connection, e.g. "how many bytes were sent". You have 68. |
| **Label** | The answer: `0` = normal (BENIGN), `1` = attack. |
| **Training set** | Rows the model learns from (60% of the data, 267,984 rows). |
| **Validation set** | Rows used to *choose settings*, e.g. which confidence cut-off. (20%) |
| **Test set** | Rows used **once**, at the very end, to report the final score. (20%) Never look at it while tuning. |
| **Label budget** | The fraction of training rows whose labels you are allowed to keep: 1%, 5%, 10%. |
| **Unlabelled pool** | The remaining training rows with their labels thrown away (but kept in a locked drawer so you can check yourself later). |
| **Stratified** | When you take a slice, keep the same mix of classes. A plain random 1% slice could contain zero examples of a rare attack. |
| **Seed** | A number that fixes all "randomness" so a re-run gives identical results. Ours is **42**. |
| **Pseudo-labelling** | Train on the few labels → let the model guess the unlabelled rows → keep only the guesses it is very sure about → add them as if they were real labels → retrain. |
| **Confidence** | The probability the model attaches to its guess, 0.0–1.0. `predict_proba` gives it. |
| **Co-training** | Split the 68 features into two halves, train two models that see different halves, and let each one hand its most confident guesses to the other. |
| **Accuracy** | Fraction of rows called correctly. **Misleading here**: 85% of rows are normal, so a model that says "normal" to everything scores 85%. |
| **Macro-F1** | Score computed separately for normal and for attack, then averaged. A do-nothing model scores badly. **This is the metric to judge on.** |
| **Recall (attack)** | Of all real attacks, what fraction did we catch? Low recall = attacks walked straight past you. |
| **ROC-AUC** | How well the confidence scores rank attacks above normal traffic, 0.5 = coin flip, 1.0 = perfect. |
| **FAR (false alarm rate)** | Of all genuinely normal traffic, what fraction did we wrongly flag? `FP / (FP + TN)`. Analysts drown at high FAR. |
| **Ablation** | Change exactly one thing, keep everything else identical, report before/after. It shows which choice actually mattered. |
| **Leakage** | Accidentally letting information from the test set (or from hidden labels) reach the model. It inflates your score and invalidates the lab. |

---

## 2. What already exists (do not rebuild it)

Lab 1 is finished and sits in `collab/collab/`. Lab 2 **reuses it** — that is explicitly graded.

```
collab/collab/
├── src/config.py            SEED = 42, SAMPLE_FRACTION = 0.20, all paths
├── src/clean.py             produced data/processed/clean.csv
├── src/prepare.py           produced data/processed/splits.joblib   <- the gold
├── src/metrics.py           accuracy, macro_f1, recall_attack, roc_auc, FAR  <- reuse as-is
├── src/train_binary.py      LogisticRegression / RandomForest / MLP
└── results/tables/binary_results.csv                                 <- the upper line
```

`splits.joblib` (512 MB) is a Python dictionary containing the **already-split, already-scaled**
data. Load it with `joblib.load(...)` and you get these keys:

| Key | What it holds |
|---|---|
| `X_train`, `X_val`, `X_test` | Raw features (DataFrames). Use these for Random Forest. |
| `X_train_s`, `X_val_s`, `X_test_s` | Scaled features (arrays). Use these for Logistic Regression / MLP. |
| `yb_train`, `yb_val`, `yb_test` | Binary labels, `0` normal / `1` attack. **Lab 2 uses these.** |
| `ym_train`, `ym_val`, `ym_test` | Attack-type labels ("DoS Hulk", …). Use for stratifying, and for analysis. |
| `feature_names` | The 68 column names. |
| `scaler`, `seed` | The fitted StandardScaler, and `42`. |

**The dataset in numbers** (CICIDS2017, all 8 days, 20% sample, after Lab 1 cleaning):

- 446,641 rows × 68 features; 84.9% BENIGN, 15.1% attack.
- Train 267,984 / val 89,328 / test 89,329 rows.
- So the budgets mean: **1% ≈ 2,680 labelled rows**, **5% ≈ 13,399**, **10% ≈ 26,798**.

**Lab 1 upper line** (test set, binary, 100% of labels):

| Model | accuracy | macro-F1 | recall (attack) | ROC-AUC | FAR |
|---|---|---|---|---|---|
| Logistic Regression | 0.9721 | 0.9418 | 0.8325 | 0.9915 | 0.0032 |
| **Random Forest** | **0.9984** | **0.9968** | **0.9923** | **0.9999** | **0.0006** |
| MLP | 0.9948 | 0.9898 | 0.9825 | 0.9995 | 0.0030 |

> **Head-up, and it belongs in your report.** The Random Forest upper line is 0.9968 macro-F1,
> and 1% of the training set is still ~2,680 labelled rows — plenty for a Random Forest. So the
> lower line will already be *high*, and SSL has almost no room to improve. That is a real,
> reportable finding, not a bug. Two honest ways to make the story richer, both cheap:
> add an extra **0.1% budget (~268 rows)** where the gap is visible, and/or **also run
> Logistic Regression**, whose lower line is much weaker. Decide as a team in Task T01.

---

## 3. Ground rules (breaking one of these costs marks)

1. **Seed = 42 everywhere.** Where the lab asks for repeats, use seeds `42, 43, 44` and report the mean.
2. **The test set is sealed.** It is touched exactly once per method, at the end, to produce the final number. Never use it to pick a threshold, a model or a stopping point — that is what the validation set is for.
3. **The hidden labels stay hidden.** The unlabelled pool's true labels may be loaded *only* to measure pseudo-label quality for the report — never fed to a model.
4. **Same cleaning, same split, same test set as Lab 1.** That is the whole point: it makes Lab 1 and Lab 2 comparable.
5. **One JSON file per experiment run.** Nobody edits a shared results file, so nobody gets a merge conflict.
6. **Cite anything you copy** (scikit-learn docs, tutorials, papers). Sources are listed at the bottom of this file and in `CHANGELOG.md`.

---

## 4. Folder you are going to build

```
lab2/
├── README.md                  how to run it (Task T13)
├── requirements.txt           library list (Task T01)
├── run_all_lab2.py            runs everything in order (Task T15)
├── src/
│   ├── config.py              T01  paths, SEED, BUDGETS, model choice
│   ├── data.py                T02  loads the Lab 1 splits, writes the snapshot
│   ├── results_io.py          T03  scores a model + saves one run as JSON
│   ├── splits.py              T04  builds the 1% / 5% / 10% label masks
│   ├── baseline_upper.py      T05  copies the Lab 1 full-label numbers in
│   ├── baseline_lower.py      T06  few-labels-only baseline
│   ├── pseudo_label.py        T07  REQUIRED method
│   ├── cotraining.py          T08  second method
│   ├── ablation_threshold.py  T09  ablation (a): confidence cut-off
│   ├── ablation_pool.py       T10  ablation (b): 25/50/100% of the pool
│   ├── aggregate.py           T11  all runs -> one results table
│   └── plot_curve.py          T12  the metric-vs-budget curve
├── data/
│   ├── lab2_data.npz          T02  the frozen data snapshot (gitignored)
│   ├── data_fingerprint.json  T02  proof both of you use identical data
│   └── splits/                T04  budget_{b}_seed{s}.npz  (gitignored)
├── results/
│   ├── runs/*.json            one file per experiment run
│   ├── tables/                the final table
│   └── figures/               the curve
└── report/                    the 2–3 page PDF
```

---

## 5. Who does what

| Person A (SSL track 1) | Person B (SSL track 2) |
|---|---|
| T01 skeleton + config **(blocking, do first)** | T00 environment + get the Lab 1 folder |
| T02 data snapshot | T03 scoring + results writer |
| T05 upper baseline | T04 label-budget splits |
| **T07 pseudo-labelling (required)** | T06 lower baseline |
| T09 ablation (confidence cut-off) | **T08 co-training** |
| T11 results table | T10 ablation (pool size, optional) |
| T14 report: intro, method, discussion | T12 curve figure |
| | T13 README |
| T15 reproducibility check (together) | T15 reproducibility check (together) |
| T16 submission (together) | T16 submission (together) |

**Dependency map** — anything on the same line can happen simultaneously:

```
T00 (B)  ──┐
T01 (A)  ──┴──> T02 (A)  ─────────────┐
              > T03 (B)  ──┐          │
                           ├─> T04 (B)┤
                           │          ├─> T05 (A)   T06 (B)
                           │          │      └──> T07 (A)   T08 (B)      <- the two big ones, fully parallel
                           │          │              └─> T09 (A)   T10 (B)
                           │          │                     └──> T11 (A) ──> T12 (B) ──> T14 (A) + T13 (B)
                           └──────────┘                                              └──> T15 ──> T16
```

Only **T01 → T02/T03 → T04** is a real bottleneck, and it is about two hours of work in total.
After that, A and B never touch the same file until T11.

---

# 6. The tasks

Each task is written so you can do it without reading the others. Copy the "Done when"
checklist into your commit message.

---

## T00 — Get a working machine (Person B, but both must end up here)

**Needs:** nothing. **Produces:** a Python that can `import sklearn`. **Time:** 30 min.

**Why:** Person B's clone of the repo will *not* contain `collab/` — it is 1.6 GB and
`.gitignore` blocks it. Without it there is no data, and no Lab 2.

**Steps**

1. Clone the repo, then create an isolated Python environment so this lab cannot break your other projects:
   ```bash
   cd AI-for-Cybersecurity-Lab2
   python3 -m venv .venv
   source .venv/bin/activate          # Windows: .venv\Scripts\activate
   pip install pandas scikit-learn matplotlib joblib tabulate
   ```
2. Get the Lab 1 folder from Person A. It is too big to email — use the OneDrive folder already
   linked in `collab/collab/README.md`, or a USB stick. You need **at minimum**:
   - `collab/collab/data/processed/splits.joblib` (512 MB) — this alone is enough for all of Lab 2, or
   - `collab/collab/data/processed/clean.csv` (150 MB) + run `python src/prepare.py` to rebuild `splits.joblib` yourself (5 min, identical output because the seed is fixed).
3. Sanity check:
   ```bash
   python -c "import joblib; d=joblib.load('collab/collab/data/processed/splits.joblib'); print(d['X_train'].shape, d['yb_train'].mean(), d['seed'])"
   ```
   You should see `(267984, 68) 0.1507... 42`. If the numbers differ, stop — you and Person A are
   working on different data and nothing will be comparable.

**Done when:** the sanity check prints those numbers on **both** machines.

**Sources:** [Python venv guide](https://docs.python.org/3/library/venv.html), Lab 1 `README.md`.

---

## T01 — Lab 2 skeleton and central settings (Person A) — **blocking, do this first**

**Needs:** nothing. **Produces:** `lab2/` folders, `lab2/src/config.py`, `lab2/requirements.txt`. **Time:** 30 min.

**Why:** Every later script reads its settings from one file. When your report says "we used a
confidence cut-off of 0.95 and seed 42", the marker can open one file and see it. It is also what
lets two people work apart: you both code against the same constants.

**Steps**

1. Create the folders shown in section 4 (`lab2/src`, `lab2/data/splits`, `lab2/results/runs`,
   `lab2/results/tables`, `lab2/results/figures`, `lab2/report`).
2. Write `lab2/src/config.py`. It must define, with a comment on each:
   ```python
   SEED = 42                      # same as Lab 1
   SEEDS = [42, 43, 44]           # repeats, because at 1% the exact rows matter a lot
   BUDGETS = [0.01, 0.05, 0.10]   # add 0.001 if the team agrees (see the head-up in section 2)
   BASE_MODEL = "RandomForest"    # the one model used by EVERY method, so comparisons are fair
   CONF_THRESHOLD = 0.95          # pseudo-label confidence cut-off (T07)
   MAX_ITERS = 3                  # how many pseudo-labelling rounds
   LAB1_ROOT = Path(...)          # ../collab/collab  -- overridable with an env var
   ```
   plus the output paths, and a `mkdir(parents=True, exist_ok=True)` loop at the bottom
   (copy that pattern straight from `collab/collab/src/config.py`).
3. **Team decision to record in the file as a comment:** one base model for everything.
   Random Forest is the recommendation — it was Lab 1's winner, needs no scaling, gives
   `predict_proba`, and trains in seconds. Optionally repeat the whole lab with Logistic
   Regression later; it makes the SSL gains far more visible.
4. Write `lab2/requirements.txt`: `pandas`, `scikit-learn>=1.3`, `numpy`, `matplotlib`, `joblib`, `tabulate`.

**Done when:** `python -c "import sys; sys.path.insert(0,'lab2/src'); import config; print(config.SEED, config.BUDGETS)"` prints `42 [0.01, 0.05, 0.1]`.

**Sources:** `collab/collab/src/config.py` (this is a direct reuse).

---

## T02 — Freeze the data snapshot (Person A)

**Needs:** T01. **Produces:** `lab2/data/lab2_data.npz`, `lab2/data/data_fingerprint.json`, `lab2/src/data.py`. **Time:** 45 min.

**Why:** `splits.joblib` is 512 MB and holds things Lab 2 does not need. Squeeze it into one
compact file with a `load_data()` helper, so every later script starts with one identical line and
nobody re-splits the data by accident. The fingerprint is a few numbers that both of you print and
compare — cheap insurance against silently working on different data.

**Steps**

1. `lab2/src/data.py` loads `LAB1_ROOT/data/processed/splits.joblib` and saves to `lab2_data.npz`:
   `X_train, X_val, X_test` (raw), `X_train_s, X_val_s, X_test_s` (scaled), `yb_*` (binary labels),
   `ym_*` (attack names, as strings — needed for stratifying), `feature_names`.
2. Add `load_data()` that returns a dict, so every other script begins:
   ```python
   from data import load_data
   d = load_data()
   ```
3. Write `data_fingerprint.json`: row counts of each split, number of features, attack rate per
   split rounded to 6 decimals, and the first 5 feature names. Print it too.
4. **Do not re-split, re-scale or re-clean anything.** Reusing Lab 1's exact split is a graded requirement.

**Done when:** the fingerprint file exists and reads `train 267984 / val 89328 / test 89329`,
`68 features`, `attack rate ≈ 0.1507` in all three splits — and Person B gets byte-identical numbers.

**Sources:** `collab/collab/src/prepare.py`, [`numpy.savez_compressed`](https://numpy.org/doc/stable/reference/generated/numpy.savez_compressed.html).

---

## T03 — Scoring and the results writer (Person B)

**Needs:** T01 only — **not** the data, so you can do this while Person A is still copying files. **Produces:** `lab2/src/results_io.py`. **Time:** 30 min.

**Why:** This is the contract that keeps you two independent. Every experiment, whoever runs it,
writes **one JSON file per run** into `lab2/results/runs/`. Nothing is ever appended to a shared
CSV, so you can both run experiments all afternoon and never hit a merge conflict. The final table
(T11) is just "read every JSON in that folder".

**Steps**

1. Reuse Lab 1's metric code rather than rewriting it — that is graded. At the top of the file:
   ```python
   sys.path.insert(0, str(config.LAB1_ROOT / "src"))
   from metrics import evaluate_binary, false_alarm_rate   # Lab 1 code, unchanged
   ```
2. Write `score_and_save(method, budget, seed, model, X_test, y_test, extra=None)` which:
   calls `model.predict` and `model.predict_proba(...)[:, 1]`, passes them to Lab 1's
   `evaluate_binary`, and writes JSON with **exactly** these keys:
   ```json
   {"method": "pseudo_label", "budget": 0.01, "seed": 42, "base_model": "RandomForest",
    "accuracy": 0.0, "macro_f1": 0.0, "recall_attack": 0.0, "roc_auc": 0.0, "FAR": 0.0,
    "n_labeled": 2680, "extra": {"threshold": 0.95, "n_pseudo_added": 41234, "rounds": 3}}
   ```
   File name: `{method}_b{budget}_s{seed}.json` (e.g. `pseudo_label_b0.01_s42.json`).
   `extra` is a free-form dict — put anything method-specific in there, so neither of you ever
   needs to change this file again.
3. Add a `main()` that runs it on fake random arrays, so you can test it with no data at all.

**Done when:** `python lab2/src/results_io.py` writes a valid demo JSON and prints it. Delete the demo file afterwards.

**Sources:** `collab/collab/src/metrics.py`, [scikit-learn metrics guide](https://scikit-learn.org/stable/modules/model_evaluation.html).

---

## T04 — Build the "few labels" splits (Person B)

**Needs:** T01 (T02 to run for real; develop against fake data meanwhile). **Produces:** `lab2/src/splits.py` and `lab2/data/splits/budget_{b}_seed{s}.npz`. **Time:** 1 h.

**Why:** This is the heart of the experiment: pretending you only have a few labels. You pick a
small stratified slice of the *training* set to keep labels for, and hide the labels on everything
else. Do it wrong — sample without stratifying, or slice the validation/test set by mistake — and
every result afterwards is meaningless.

**Steps**

1. For each budget in `BUDGETS` and each seed in `SEEDS`, use
   [`train_test_split`](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.train_test_split.html)
   with `train_size=budget`, `stratify=ym_train`, `random_state=seed` — stratify on the **attack-type**
   labels, not the binary ones, so rare attacks like SSH-Patator survive in the slice.
   You only need the row indices, so split `np.arange(len(y_train))`.
2. Save the two index arrays: `labeled_idx` and `unlabeled_idx` (their union must be the whole
   training set, and they must not overlap).
3. Print a small report per budget: how many labelled rows, the attack rate in the slice
   (should stay ≈ 0.1507), and how many attack *types* made it in. At 1% some tiny classes will
   have 0 or 1 rows — note it, it belongs in the report's discussion.
4. Add `load_split(budget, seed)`.

**Common trap:** the "unlabelled pool" means *labels you pretend not to have*. Keep the true labels
in the arrays; just never pass them to a model. T07/T08 will use them only to measure how good the
pseudo-labels were.

**Done when:** all 9 (or 12) `.npz` files exist; `labeled_idx` sizes are 2,680 / 13,399 / 26,798;
attack rate in every slice is within ±0.005 of 0.1507; `set(labeled) & set(unlabeled) == empty`.

**Sources:** [scikit-learn: stratified splitting](https://scikit-learn.org/stable/modules/cross_validation.html#stratification), lab brief Step 2.

---

## T05 — The upper line (Person A)

**Needs:** T02, T03. **Produces:** `lab2/results/runs/upper_*.json`, `lab2/src/baseline_upper.py`. **Time:** 30 min.

**Why:** The upper line is "what could I have scored if I had paid for every label" — the ceiling
SSL is trying to reach. Lab 1 already computed it; the point of this task is to get those numbers
into the same JSON format as everything else, so T11 can build one table.

**Steps**

1. Read `collab/collab/results/tables/binary_results.csv` and re-emit the Random Forest row (and,
   if you are also running Logistic Regression, that row) as `upper_b1.0_s42.json` with
   `method="upper_full_labels"`, `budget=1.0`.
2. **Verification (recommended, 2 minutes of compute):** retrain the same Random Forest
   (`n_estimators=300, max_depth=20, random_state=42`) on the full training set through your own
   T03 writer and check you reproduce macro-F1 ≈ 0.9968. If it matches, your Lab 2 plumbing is
   proven correct against a known-good number. If it does not, fix it now — before it silently
   corrupts every SSL result.

**Done when:** the JSON exists and shows macro-F1 0.9968, FAR 0.0006; the retrain check matches to ~3 decimals.

**Sources:** `collab/collab/results/tables/binary_results.csv`, `collab/collab/src/train_binary.py`.

---

## T06 — The lower line: few labels only (Person B)

**Needs:** T02, T03, T04. **Produces:** `lab2/results/runs/lower_*.json`, `lab2/src/baseline_lower.py`. **Time:** 1 h.

**Why:** The single most important comparison in the lab, and the one students most often skip.
It answers: "what do I get from the few labels alone, with no clever tricks?" Any SSL result that
does not beat this line means SSL added nothing but complexity. The lab brief lists skipping this
as the #1 common mistake.

**Steps**

1. For every budget × seed: load the split, take `X_train[labeled_idx]` and `y_train[labeled_idx]`,
   train the base model with `random_state=SEED`, and score on the **test set** via T03.
   The unlabelled pool is not used at all here — that is the point.
2. Save with `method="lower_few_labels"`. That is 9 runs (12 with the extra budget); each takes
   seconds to a couple of minutes.
3. Print a table of mean macro-F1 per budget so you can see the trend before anything else exists.

**Done when:** 9 JSON files exist and macro-F1 rises with the budget (e.g. 1% < 5% < 10% < 0.9968).

**Sources:** lab brief Step 1, `collab/collab/src/train_binary.py`.

---

## T07 — Pseudo-labelling — **REQUIRED METHOD** (Person A)

**Needs:** T02, T03, T04. **Produces:** `lab2/results/runs/pseudo_label_*.json`, `lab2/src/pseudo_label.py`. **Time:** 2–3 h.

**Why:** The simplest semi-supervised idea that works. Train on the few labels you have; the model
is weak but not useless, so let it guess the unlabelled rows; some guesses it makes with 99%
confidence, and those are *probably* right; treat those as if a human had labelled them, and
retrain on the bigger set. Repeat. Introduced for neural networks by Lee (2013), and known more
generally as **self-training**.

The whole method lives or dies on the **confidence cut-off**. Set it too low and the model happily
learns from its own mistakes, drifting further from the truth every round — this is *confirmation
bias*, and the lab brief lists it as common mistake #2.

**Steps**

1. **Round 0:** train the base model on the labelled slice only (identical to T06 — you can even
   reuse it as your starting point).
2. **Each round (up to `MAX_ITERS = 3`):**
   - `proba = model.predict_proba(X_train[unlabeled_idx])`
   - `confidence = proba.max(axis=1)`, `guess = proba.argmax(axis=1)`
   - keep rows where `confidence >= CONF_THRESHOLD` (start at 0.95)
   - move those rows out of the pool and into the labelled set **with the model's guess as the label** — never the true label
   - retrain from scratch on `labelled slice + accepted pseudo-labels`
   - stop early if a round accepts 0 new rows
3. **Record per round** (this is what makes your report good): rows accepted, cumulative pool
   consumed, and — for reporting only — the **pseudo-label accuracy**, i.e. how many accepted
   guesses matched the hidden true label. That number is the difference between "SSL worked" and
   "SSL worked and I can explain why".
4. Score the final model on the test set through T03, once. Put rounds, threshold, total pseudo-labels
   added and pseudo-label accuracy into `extra`.
5. Run all budgets × all seeds.

**Design decisions to state explicitly in the report:** the cut-off (0.95), the number of rounds (3),
whether you retrain from scratch or warm-start (from scratch is simpler and safer), and whether you
cap how many rows may be added per round.

**Shortcut worth knowing:** scikit-learn has
[`SelfTrainingClassifier`](https://scikit-learn.org/stable/modules/generated/sklearn.semi_supervised.SelfTrainingClassifier.html),
which does exactly this loop with `threshold=` and unlabelled rows marked as `-1`. Writing the loop
yourself gives you the per-round diagnostics above, which are worth more marks — but running the
library version too, as a cross-check that your numbers agree, is a strong move. (In scikit-learn
≥ 1.6 the first argument is `estimator=`; it was `base_estimator=` before.)

**Done when:** 9 JSON files with `method="pseudo_label"`; each has a per-round log in `extra`;
you can state in one sentence how the score compares with T06's lower line at each budget.

**Sources:** Lee, D.-H. (2013), *Pseudo-Label: The Simple and Efficient Semi-Supervised Learning
Method for Deep Neural Networks*, ICML Workshop on Challenges in Representation Learning ·
[scikit-learn: semi-supervised learning](https://scikit-learn.org/stable/modules/semi_supervised.html) ·
Van Engelen & Hoos (2020), *A survey on semi-supervised learning*, Machine Learning 109(2), 373–440.

---

## T08 — Co-training — **SECOND METHOD** (Person B)

**Needs:** T02, T03, T04. **Produces:** `lab2/results/runs/cotraining_*.json`, `lab2/src/cotraining.py`. **Time:** 2–3 h.

**Why:** Pseudo-labelling has one blind spot: a model that is confidently wrong stays confidently
wrong, because it only ever hears its own opinion. Co-training (Blum & Mitchell, 1998) fixes that by
training **two** models on two different halves of the features — two "views" of the same
connection, e.g. one sees packet-size statistics, the other sees timing statistics. Each model
teaches the other with its most confident guesses. Because they see different evidence, one can
correct the other's blind spot.

Chosen over Mean Teacher and FixMatch because it needs no neural network and no data augmentation —
and "how do you meaningfully augment a row of a spreadsheet?" is an unsolved question the lab brief
itself flags. **Say exactly that in the report** as the justification for your choice.

**Steps**

1. **Split the 68 features into two views.** Simplest defensible choice: shuffle the feature indices
   with `random_state=SEED` and cut in half; record which features landed in which view. A more
   interesting choice is splitting by meaning (forward/backward direction, or size/timing features)
   — either is fine if you justify it.
2. Train model A on `X[labeled, view_A]`, model B on `X[labeled, view_B]`.
3. **Each round (3 rounds is plenty):**
   - each model predicts the pool through its own view
   - take each model's top-`k` most confident rows (e.g. `k = 500` per class, or everything above 0.95)
   - **add model A's confident guesses to model B's labelled set, and vice versa** — the cross-over is the whole point
   - retrain both
4. **Final prediction:** average the two models' probabilities (each still using its own view), then
   threshold at 0.5. Score once on the test set via T03.
5. Log per round in `extra`: rows exchanged in each direction, and the accuracy of the exchanged
   labels against the hidden truth (report only).

**Watch out:** a view must be genuinely informative on its own. If one half of the features is
useless, that model poisons the other. Report each view's solo macro-F1 — it is a one-line addition
and makes the discussion much stronger.

**Done when:** 9 JSON files with `method="cotraining"`; `extra` records the view split, rounds and
exchange counts; you can compare against T06 at each budget.

**Sources:** Blum, A. & Mitchell, T. (1998), *Combining Labeled and Unlabeled Data with Co-Training*,
COLT '98 · Van Engelen & Hoos (2020) · lab brief section 5.

---

## T09 — Ablation (a): does the confidence cut-off matter? (Person A) — **the lab requires one ablation**

**Needs:** T07. **Produces:** `lab2/results/runs/abl_thresh_*.json`, `lab2/src/ablation_threshold.py`. **Time:** 1 h.

**Why:** An ablation changes exactly one setting and holds everything else still, so you can say
*this specific choice* caused *this specific change*. The cut-off is the obvious candidate: it is
the one knob that decides whether pseudo-labelling helps or feeds the model garbage.

**Steps**

1. Re-run T07 unchanged except `CONF_THRESHOLD ∈ {0.70, 0.80, 0.90, 0.95, 0.99}`.
2. Do it at the **1% budget** (where SSL matters most) and at **10%** (to show the effect shrinks
   when labels are plentiful). Use `seed=42`, or all three seeds if it runs fast enough.
3. Save with `method="abl_threshold"` and `extra={"threshold": t}`.
4. Report before/after as a small table: threshold → pseudo-labels accepted, pseudo-label accuracy,
   test macro-F1, test FAR. The expected shape is a trade-off: a low cut-off accepts many rows but
   many are wrong; a high cut-off accepts few but clean ones. Whether accuracy of the pseudo-labels
   or their *quantity* wins is exactly what you are measuring.

**Done when:** 10 JSON files (5 thresholds × 2 budgets) and a paragraph naming the best cut-off and by how much it beat the worst.

**Sources:** lab brief Step 5, option (a).

---

## T10 — Ablation (b): how much unlabelled data do you actually need? (Person B, optional)

**Needs:** T07 or T08. **Produces:** `lab2/results/runs/abl_pool_*.json`, `lab2/src/ablation_pool.py`. **Time:** 1 h.

**Why:** The lab requires one ablation; T09 covers it. This second one is cheap and answers a
question a security team would really ask: is it worth storing and processing *all* the unlabelled
traffic, or do you get most of the benefit from a quarter of it?

**Steps**

1. Keep everything from your method fixed; use only 25% / 50% / 100% of the unlabelled pool
   (subsample it with `random_state=SEED`, stratified is not required — say which you did).
2. Run at the 1% budget, `method="abl_pool"`, `extra={"pool_fraction": f}`.
3. Report the curve: does the score plateau? If 25% of the pool gets you 95% of the gain, that is a
   genuinely useful practical finding for the discussion.

**Done when:** 3 JSON files and one sentence on whether more unlabelled data kept helping.

**Sources:** lab brief Step 5, option (b).

---

## T11 — Build the results table (Person A)

**Needs:** T05, T06, T07, T08 (T09/T10 fold in automatically). **Produces:** `lab2/results/tables/lab2_results.csv` + `.md`, `lab2/src/aggregate.py`. **Time:** 1 h.

**Why:** All the numbers exist as scattered JSON files. This turns them into the one table the
report is built around. Because it just globs a folder, it works no matter who ran what, or when.

**Steps**

1. `glob("lab2/results/runs/*.json")` → pandas DataFrame.
2. Group by `method` × `budget`, average across seeds, and report **mean ± standard deviation** for
   accuracy, macro-F1, recall, ROC-AUC and FAR. The spread across seeds matters: at 1% it may be
   larger than the gain from SSL, and if so, say so — that is a genuine finding about how fragile
   tiny label budgets are.
3. Add a `delta_macro_f1_vs_lower` column: the SSL score minus the lower line at the same budget.
   This is the number the marker is looking for.
4. Write both a CSV and a markdown table (`tabulate`) you can paste into the report.
5. Order rows: lower line → pseudo-label → co-training → upper line, so the table reads as a story.

**Done when:** one table exists with a row per method × budget, and every cell the lab asked for
(accuracy, macro-F1, recall, ROC-AUC, FAR) is present.

**Sources:** lab brief Step 6, `collab/collab/src/compare.py` (same idea, reuse the layout).

---

## T12 — The curve (Person B)

**Needs:** T11. **Produces:** `lab2/results/figures/macro_f1_vs_budget.png`, `lab2/src/plot_curve.py`. **Time:** 45 min.

**Why:** The lab asks for one curve of a metric against the label budget, with both reference lines
drawn on it. Done right, one glance answers the whole lab question: how much of the full-label score
do you keep, and at what budget.

**Steps**

1. X axis: label budget (1 / 5 / 10 %) — a log scale reads better if you added the 0.1% budget.
   Y axis: macro-F1.
2. Draw: the lower line (few labels only), pseudo-labelling, co-training, plus a **horizontal dashed
   line** for the Lab 1 upper line, labelled with its value.
3. Error bars = standard deviation across the three seeds.
4. Because the scores all sit near 0.99, zoom the Y axis (e.g. 0.90–1.00) — but say in the caption
   that you did, so nobody thinks you are exaggerating a gap.
5. Make a second figure for **FAR vs budget**. It is 5 extra lines of code and FAR is one of the two
   metrics the brief tells you to judge on.
6. Write a real caption ("Figure 1: macro-F1 against label budget, mean of 3 seeds…") and make sure
   the report text refers to it by number — that is an explicit grading criterion.

**Done when:** both PNGs exist at `dpi=150`, are readable, and every line is in the legend.

**Sources:** [matplotlib errorbar](https://matplotlib.org/stable/api/_as_gen/matplotlib.pyplot.errorbar.html), `collab/collab/src/compare.py`.

---

## T13 — README (Person B)

**Needs:** T01–T12 roughly working. **Produces:** `lab2/README.md`. **Time:** 45 min.

**Why:** Graded directly ("a README explains how to run it; seed is fixed"). A marker who cannot run
your code marks what they can see.

**Steps** — cover: what the lab does; the library list and how to install it; **how to get the Lab 1
`splits.joblib`** and where to put it; the exact commands in order; expected runtime; where the
outputs land; **the seed (42) and the seeds list**; a table mapping each script to the lab step it
implements (copy the style from Lab 1's README); and a short troubleshooting section.

**Done when:** Person A can follow it on a clean checkout without asking a single question.

**Sources:** `collab/collab/README.md`.

---

## T14 — The report, 2–3 pages (Person A writes the frame, both write their own method)

**Needs:** T11, T12. **Produces:** `report/Lab2_report.pdf`. **Time:** 3 h. **Worth 25% of the grade.**

**Why:** The marking scheme rewards a clear report over a clever one. Keep to the brief's suggested
sections and make sure every table and figure is referred to by number in the text.

**Sections**

1. **Why labels are scarce, and what SSL is** — one paragraph, in your own words.
2. **What you did** — dataset and the reused Lab 1 cleaning/split; how the budgets were made
   (stratified, seed 42, 3 seeds); the two methods and their settings; state the confidence cut-off.
3. **Results** — the T11 table and the T12 curve, each with a caption you actually cite.
4. **Discussion** — did SSL beat the lower line, at which budget, by how much; where it failed;
   what the pseudo-label accuracy told you; whether the seed-to-seed spread swamps the gain.
   **Explicitly discuss the ceiling problem**: with a 0.9968 upper line and 2,680 labelled rows at
   1%, the headroom for SSL is tiny — a finding, not a failure.
5. **Who did what** — one or two honest lines. Required.
6. **Appendix** — screenshots of each code part with a comment explaining it (the brief asks for
   this by name; do not skip it, it is inside the 25%).

**Done when:** exported to PDF, 2–3 pages plus appendix, every figure numbered and cited, sources listed.

**Sources:** lab brief section 7 and the grading table.

---

## T15 — Reproducibility check (both, together, 1 h)

**Needs:** everything. **Produces:** `lab2/run_all_lab2.py`, a clean end-to-end run.

**Why:** "Code runs start-to-finish and reproduces your numbers" is the first line of the 30%
implementation criterion. The most common failure is a script that only works because of a variable
left over in someone's notebook.

**Steps**

1. Write `run_all_lab2.py` in Lab 1's style: a list of `(title, script)` pairs run with
   `subprocess.run`, stopping at the first failure.
2. Delete `lab2/results/runs/` and `lab2/data/splits/`, then run it from a fresh terminal on **both**
   machines.
3. Compare the final tables. Same seed → same numbers, to the decimal. If they differ, hunt down the
   unseeded randomness before doing anything else.
4. Check the sealed-test-set rule one last time: grep for `X_test` and confirm it appears only in
   final scoring calls, never in a fitting or threshold-selection step.

**Done when:** one command reproduces every number in the report, on both machines.

**Sources:** `collab/collab/run_all.py`.

---

## T16 — Submit (both, 30 min)

**Steps:** update `CHANGELOG.md` with the last entries · commit and push · export the report to PDF ·
zip the repo *without* `data/` and `collab/` (the `.gitignore` already lists them) · upload the code
(zip or repo link) **and** the PDF to Canvas before the deadline · confirm the "who did what" line is
in the report.

**Done when:** both files are on Canvas and you both have the submission receipt.

---

# 7. Common mistakes (from the brief, plus two of our own)

1. **Skipping the few-labels-only baseline (T06)** — then you cannot show SSL helped at all.
2. **Confidence cut-off too low** — the model learns from its own bad guesses and drifts.
3. **Judging by accuracy alone** — 85% of rows are BENIGN, so 85% accuracy means "detected nothing".
   Judge on macro-F1 and FAR, and always check recall.
4. **Peeking at the test set** — choosing the cut-off or the round count by test score invalidates
   the result. Use the validation set.
5. **Leaking the hidden labels** — the true labels of the unlabelled pool may be used for *measuring*
   pseudo-label quality, never for training.
6. **Re-splitting the data in Lab 2** — the split must be Lab 1's, byte for byte, or the two labs
   cannot be compared.

---

# 8. Sources

**Lab material**
- Lab 2 brief: `2. Lab2_Learning_when_labels_are_scarce.pdf`
- Lab 1 brief and code: `collab/Lab1_Building_Your_First_Intrusion_Detector.pdf`, `collab/collab/`

**Method papers**
- Lee, D.-H. (2013). *Pseudo-Label: The Simple and Efficient Semi-Supervised Learning Method for Deep Neural Networks.* ICML Workshop on Challenges in Representation Learning.
- Blum, A. & Mitchell, T. (1998). *Combining Labeled and Unlabeled Data with Co-Training.* COLT '98, 92–100.
- Van Engelen, J. E. & Hoos, H. H. (2020). *A survey on semi-supervised learning.* Machine Learning, 109(2), 373–440.
- Tarvainen, A. & Valpola, H. (2017). *Mean teachers are better role models.* arXiv:1703.01780. (Not used — needs a neural network.)
- Sohn, K. et al. (2020). *FixMatch: Simplifying Semi-Supervised Learning with Consistency and Confidence.* arXiv:2001.07685. (Not used — augmenting tabular data is ill-defined.)

**Library documentation**
- scikit-learn, *Semi-supervised learning* user guide — https://scikit-learn.org/stable/modules/semi_supervised.html
- scikit-learn, `SelfTrainingClassifier` — https://scikit-learn.org/stable/modules/generated/sklearn.semi_supervised.SelfTrainingClassifier.html
- scikit-learn, `train_test_split` — https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.train_test_split.html
- scikit-learn, *Metrics and scoring* — https://scikit-learn.org/stable/modules/model_evaluation.html
- matplotlib — https://matplotlib.org/stable/

**Dataset**
- Sharafaldin, I., Lashkari, A. H. & Ghorbani, A. A. (2018). *Toward Generating a New Intrusion Detection Dataset and Intrusion Traffic Characterization.* ICISSP 2018.
- CICIDS2017 — https://www.unb.ca/cic/datasets/ids-2017.html
- CICIDS2018 — https://www.unb.ca/cic/datasets/ids-2018.html
