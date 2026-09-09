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

- [ ] `T00` environment set up, Lab 1 `splits.joblib` present on both machines — @B
- [ ] `T01` `lab2/` skeleton and `src/config.py` (SEED, BUDGETS, base model) — @A
- [ ] `T02` frozen data snapshot + fingerprint — @A
- [ ] `T03` scoring helper and per-run JSON writer — @B
- [ ] `T04` stratified 1% / 5% / 10% label splits — @B
- [ ] `T05` upper line (Lab 1, 100% labels) — @A
- [ ] `T06` lower line (few labels only) — @B
- [ ] `T07` pseudo-labelling (required method) — @A
- [ ] `T08` co-training (second method) — @B
- [ ] `T09` ablation: confidence cut-off — @A
- [ ] `T10` ablation: unlabelled pool size (optional) — @B
- [ ] `T11` results table — @A
- [ ] `T12` macro-F1 and FAR curves — @B
- [ ] `T13` `lab2/README.md` — @B
- [ ] `T14` report PDF (2–3 pages + appendix) — @A + @B
- [ ] `T15` reproducibility check on both machines — @A + @B
- [ ] `T16` submission to Canvas — @A + @B
