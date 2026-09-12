# Lab 2 in plain language: what happens when you barely have any labels

*Companion to `Report-Academic.md`. Same experiment, same numbers, no jargon.
Written 2026-09-12 from the verified run on Lab 1's data.*

---

## 1. The problem, in everyday terms

A network intrusion detector is a program that watches connections going in and out of a company
and tries to say which ones are attacks.

To build one with machine learning, you first need examples that somebody has already sorted into
"normal" and "attack". Those answers are called **labels**. The catch is that labels are expensive.
A human security analyst has to sit and look at connections one at a time and decide what each one
is. For millions of connections, nobody can afford that.

So in real life you end up with a small pile of connections somebody labelled, and a mountain of
connections nobody has looked at.

**The question of this lab:** can the mountain still be useful, even without answers attached?

The family of methods that tries to use it is called **semi-supervised learning**. The idea is
almost cheeky: let the detector look at the unlabelled mountain, write its own answers on the ones
it feels sure about, and then learn from those too, as if a human had written them.

## 2. What we actually did

We already had a working detector from Lab 1, trained with **all** the labels. For Lab 2 we
pretended we had almost none.

- We took Lab 1's exact data and kept its exact division into practice material and the final exam.
- We hid labels on purpose, keeping only **1%**, **5%** or **10%** of them. One percent means 2,680
  labelled connections, and about 265,000 with the answers hidden.
- We then compared three things at each level, which is the part that makes the answer trustworthy:
  1. **The plain detector**: train on the few labels, ignore the mountain. This is the bar to beat.
     If a clever method can't beat this, the cleverness bought nothing.
  2. **Pseudo-labelling**: the detector guesses the mountain, keeps only guesses it is at least 95%
     sure about, and re-trains with those guesses included.
  3. **Co-training**: split the measurements in half, so two detectors each see a different half of
     the evidence. Each one passes its most confident guesses to the other. Because they look at
     different clues, one can catch what the other misses.
- Everything was run three times with different random draws of "which labels do we keep", because
  when you only keep 1%, *which* 1% you happen to get matters a lot.
- The final exam was never used for tuning. It was used once, at the end, to produce the scores.

## 3. How we measure "good"

Accuracy is a trap here. About 85% of the traffic is normal, so a lazy detector that shouts
"normal!" at everything is 85% accurate and catches nothing. We therefore watch three things:

- **Macro-F1**: scores normal traffic and attacks separately and averages them, so ignoring attacks
  is punished. This is the headline number.
- **Recall**: of all the real attacks, how many did we catch? Missing attacks is the expensive kind
  of mistake.
- **False alarm rate (FAR)**: of all the genuinely normal traffic, how much did we wrongly flag?
  High FAR means analysts drown in false alerts and start ignoring the tool.

## 4. What happened

Headline numbers, higher is better except FAR:

| How many labels we kept | Plain detector | Pseudo-labelling | Co-training |
|---|---|---|---|
| **1%** (2,680 labels) | 0.9795 | 0.9814 | **0.9833** |
| **5%** (13,399 labels) | 0.9940 | **0.9942** | 0.9938 |
| **10%** (26,798 labels) | 0.9943 | **0.9947** | 0.9946 |

For reference, Lab 1 with *every* label scored 0.9968.

**The short version: semi-supervised learning helped, but only when labels were really scarce.**

At 1%, both methods beat the plain detector, and co-training was best. It also cut false alarms by
more than half: from about 45 false alarms per 10,000 normal connections down to about 21. For a
team that actually reads the alerts, that is the most useful thing in this report.

At 5% and 10% the differences almost vanish. Once you have 13,000 labelled examples, the detector
already knows nearly everything the unlabelled mountain could teach it, so the clever methods have
nothing left to add. That is not a failure. It is a useful finding: **spend your effort on
semi-supervised learning when labelling is hardest, not when you already have plenty.**

## 5. We also tested the "how sure is sure enough?" dial

The whole method rests on one setting: how confident must the detector be before we keep its guess?
We had it at 95%. So we tried loosening it to 70% and tightening it to 99%, changing nothing else,
and scored the results on practice data rather than the final exam.

| How sure must it be? | Guesses it kept | Share of the mountain | How many were right | Score |
|---|---|---|---|---|
| Nothing kept (starting point) | 0 | 0% | — | 0.9824 |
| 70% sure | 265,244 | 99.98% | 99.10% | 0.9818 |
| 80% sure | 265,007 | 99.89% | 99.13% | 0.9812 |
| 90% sure | 264,828 | 99.82% | 99.18% | 0.9821 |
| 95% sure | 263,726 | 99.40% | 99.24% | 0.9817 |
| 99% sure | 260,633 | 98.24% | 99.38% | 0.9795 |

Two things jump out.

**Being stricter did make the guesses cleaner** — from 99.10% correct up to 99.38%. That is the dial
working as intended.

**But no setting actually helped, and the strictest one was worst.** The reason is in the third
column: the detector is so confident about this data that even demanding "99% sure" still keeps 98%
of the mountain. The dial never really tightens anything. And once you accept 265,000 guesses of
which about 1% are wrong, that is roughly 2,400 bad examples, and they are not random mistakes. They
are the detector's own blind spots, so it teaches itself to keep making them. That is the classic
trap of this method.

**So what actually made it work earlier?** The other limit. In the main experiment we also capped
*how many* guesses could be added, to twice the number of real labels. That cap, not the confidence
dial, is what turned pseudo-labelling into an improvement. Keeping a small number of the very best
guesses helps; swallowing the whole mountain does not, however careful you are about it.

## 6. The honest caveats

**The result wobbles.** At 1%, we ran three different random draws of which labels to keep. In one
of them, both clever methods made things slightly *worse*:

| Random draw | Plain detector | Pseudo-labelling | Co-training |
|---|---|---|---|
| Seed 42 | 0.9822 | 0.9786 | 0.9779 |
| Seed 43 | 0.9764 | 0.9844 | 0.9856 |
| Seed 44 | 0.9800 | 0.9811 | 0.9864 |

The improvement we report is the average of these. The spread between draws is about as large as
the improvement itself. If we had run only the first draw, as an earlier version of this experiment
did, we would have concluded the exact opposite. Anyone reading a single-run result should be
suspicious, including us.

**The guesses were almost perfect, and that is partly by design.** We checked the guessed labels
against the real hidden answers afterwards, purely to understand what happened. Essentially 100% of
them were right. That sounds wonderful, but it happens because we were very strict: we only kept
very confident guesses, and we also capped how many could be added per round. At the 1% budget the
detector added only 5,360 guessed examples out of 265,000 available. **We used about 2% of the
mountain.** Being less strict would use more of it, at the risk of learning from mistakes.

**We checked that the numbers are repeatable.** The whole experiment was run twice from scratch,
on different occasions. Every number in the table above came out the same to four decimal places.
So the results are stable; what wobbles is the choice of which labels you keep, described just
above, not the machinery.

**One dataset, one detector type.** Everything here is CICIDS2017 and one kind of model. The
pattern, "helps when labels are scarce, stops helping when they aren't", is the part we would
expect to carry over. The exact numbers are not.

## 7. What we would tell a security team

- If labelling is your bottleneck, semi-supervised learning is worth it, and the gain shows up as
  **fewer false alarms** more than as catching more attacks.
- Limit **how many** guesses you accept, not just how confident they must be. On our data the
  quantity limit was what made the difference; the confidence dial barely mattered.
- Don't trust a single run. Re-run with different random draws before believing an improvement.
- Once you can afford a few thousand good labels, put your effort somewhere else.

---

## Mini-glossary

| Word | What it means here |
|---|---|
| Label | The answer for one connection: normal, or attack |
| Budget | How many labels we allowed ourselves: 1%, 5% or 10% |
| Unlabelled pool | The connections whose answers we deliberately hid |
| Pseudo-label | A label the detector wrote itself, not a human |
| Confidence | How sure the detector says it is, from 0 to 1. We kept guesses at 0.95 and above |
| Macro-F1 | A score that treats "spot the attacks" and "leave normal traffic alone" as equally important |
| Recall | The share of real attacks we caught |
| False alarm rate | The share of normal traffic we wrongly flagged |
| Seed | A number that fixes the random choices, so the experiment can be repeated exactly |
