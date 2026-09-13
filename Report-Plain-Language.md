# Lab 2 in plain language: what happens when you barely have any labels

*Companion to `Report-Academic.md`. Same experiment, same numbers, no jargon.
The numbers come from the complete run on Google Colab, using Lab 1's exact data.*

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
  Before doing anything, the notebook checks the data's size and make-up, and prints a digital
  fingerprint of it, which matched the one taken from Lab 1's data.
- We hid labels on purpose, keeping only **1%**, **5%** or **10%** of them. One percent means 2,680
  labelled connections, and about 265,000 with the answers hidden.
- We then compared three things at each level, which is the part that makes the answer trustworthy:
  1. **The plain detector**: train on the few labels, ignore the mountain. This is the bar to beat.
     If a clever method can't beat this, the cleverness bought nothing.
  2. **Pseudo-labelling**: the detector guesses the mountain, keeps only guesses it is at least 95%
     sure about, and re-trains with those guesses included.
  3. **Co-training**: split the measurements in half, so two detectors each see a different half of
     the evidence. Each one passes its most confident guesses to the other. The hope is that because
     they look at different clues, one can catch what the other misses.
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

Headline numbers, higher is better:

| How many labels we kept | Plain detector | Pseudo-labelling | Co-training |
|---|---|---|---|
| **1%** (2,680 labels) | 0.9809 | 0.9836 | **0.9839** |
| **5%** (13,399 labels) | **0.9941** | 0.9938 | 0.9939 |
| **10%** (26,798 labels) | 0.9943 | **0.9950** | 0.9949 |

For reference, Lab 1 with *every* label scored 0.9968.

**The short version: semi-supervised learning improved the headline score only when labels were
really scarce, but it cut false alarms at every level.**

At 1%, both methods beat the plain detector by about the same amount, and the two are effectively
tied with each other. At 5% and 10% the headline differences almost vanish. Once you have 13,000
labelled examples, the detector already knows nearly everything the unlabelled mountain could teach
it about spotting attacks.

**False alarms are where the real benefit shows up.** On the final exam's 75,868 normal connections:

| How many labels we kept | Plain detector | Pseudo-labelling | Co-training |
|---|---|---|---|
| 1% | 288 false alarms | 176 | **148** |
| 5% | 127 | 93 | **60** |
| 10% | 143 | 95 | **62** |

At every level, both methods produced fewer false alarms than the plain detector. Co-training roughly
halved them. The trade-off at 5% and 10% is that it caught slightly fewer attacks (98.4% instead of
99.0% at 5%). For a team that actually reads the alerts, fewer false alarms is often the more useful
improvement.

## 5. We also tested the "how sure is sure enough?" dial

The whole method seems to rest on one setting: how confident must the detector be before we keep its
guess? We had it at 95%. So we tried loosening it to 70% and tightening it to 99%, changing nothing
else. For this test we also removed the limit on how many guesses could be kept, so that the dial
alone decided. We scored it on practice data rather than the final exam.

| How sure must it be? | Guesses it kept | Share of the mountain | How many were right | Score |
|---|---|---|---|---|
| Nothing kept (starting point) | 0 | 0% | — | 0.9828 |
| 70% sure | 265,142 | 99.94% | 99.16% | 0.9833 |
| 80% sure | 264,979 | 99.88% | 99.19% | 0.9823 |
| 90% sure | 264,639 | 99.75% | 99.25% | 0.9824 |
| 95% sure | 264,066 | 99.53% | 99.22% | 0.9801 |
| 99% sure | 261,064 | 98.40% | 99.41% | 0.9813 |

Three things jump out.

**Being stricter mostly made the guesses cleaner**, from 99.16% right up to 99.41%, with a small
dip at 95%. That is the dial doing roughly what it should.

**But the dial barely changes what gets kept.** The detector is so confident about this data that
even demanding "99% sure" still keeps 98% of the mountain. The dial never really tightens anything.

**And keeping almost everything did not help.** Only the loosest setting nudged the score up, by a
hair (+0.0005, smaller than the normal wobble between runs). The other four made it worse. When you
accept 265,000 guesses and about 1% are wrong, that is roughly 1,500 to 2,200 bad examples. They
are not random mistakes: they are the detector's own blind spots, so it teaches itself to keep making
them. That is the classic trap of this method.

**So what made it work in the main experiment?** Very likely the *other* limit: a cap on how many
guesses could be added, set to twice the number of real labels. We checked the logs. In the main
experiment, 94–99% of the mountain passed the 95% bar every time, so the bar kept almost everything,
and at the 1% level the cap then kept only about 1 in every 100 of those — the very best ones. Keeping a small number
of the very best guesses helped; swallowing the whole mountain did not. We say "very likely" because
this test used one random draw and practice data, while the main results used three draws and the
final exam, so it is strong evidence rather than proof.

## 6. The honest caveats

**The result wobbles.** At 1%, we ran three different random draws of which labels to keep. In one of
them, both clever methods made things slightly *worse*:

| Random draw | Plain detector | Pseudo-labelling | Co-training |
|---|---|---|---|
| Seed 42 | 0.9822 | 0.9801 | 0.9796 |
| Seed 43 | 0.9798 | 0.9844 | 0.9849 |
| Seed 44 | 0.9805 | 0.9862 | 0.9873 |

The improvement we report is the average of these, and the spread between draws is about as large as
the improvement itself. Had we run only the first draw, we would have concluded the exact opposite.
Anyone reading a single-run result should be suspicious, including us.

**Co-training didn't really work the way the idea promises.** The hope was that two detectors looking
at different clues would disagree and correct each other. In practice they never disagreed even once
about a guess they both made, across all 18 rounds. Each half of the measurements was good enough on
its own, but the two detectors were confident about largely the same things. So co-training behaved
almost like pseudo-labelling done twice, which is why the two methods ended up tied.

**The guesses were almost perfect, and that is by design.** We checked the guessed labels against the
real hidden answers afterwards, purely to understand what happened. Essentially 100% were right,
because the cap kept only the most confident ones. The cost is that at the 1% level the detector added
only 5,360 guessed examples out of 265,000 available: **it used about 2% of the mountain.**

**The software version matters about as much as the method.** We ran the whole experiment twice on
one computer and got identical numbers both times. But running the same code on the same data with
an older version of the machine-learning library, on Google Colab, moved some scores by up to 0.002.
That is about the size of the improvement we are measuring, and it even flipped whether
pseudo-labelling slightly helped or slightly hurt at the 5% level. The numbers in this report are the Colab ones, because that is the complete run
whose notebook passed every automatic check.

**One dataset, one detector type.** Everything here is CICIDS2017 and one kind of model. The pattern,
"helps when labels are scarce, stops helping when they aren't, reduces false alarms", is the part we
would expect to carry over. The exact numbers are not.

## 7. What we would tell a security team

- If labelling is your bottleneck, semi-supervised learning is worth trying, and expect the gain to
  show up as **fewer false alarms** more than as catching more attacks.
- Limit **how many** guesses you accept, not just how confident they must be. On our data the
  confidence dial barely mattered; keeping only the best handful is what helped.
- Don't trust a single run. Re-run with different random draws before believing an improvement, and
  write down your software versions.
- Once you can afford a few thousand good labels, the headline score stops improving, though false
  alarms may still drop.

---

## Mini-glossary

| Word | What it means here |
|---|---|
| Label | The answer for one connection: normal, or attack |
| Budget | How many labels we allowed ourselves: 1%, 5% or 10% |
| Unlabelled pool | The connections whose answers we deliberately hid |
| Pseudo-label | A label the detector wrote itself, not a human |
| Confidence | How sure the detector says it is, from 0 to 1. We kept guesses at 0.95 and above |
| Cap | The limit on how many guesses could be added per round: twice the number of real labels |
| Macro-F1 | A score that treats "spot the attacks" and "leave normal traffic alone" as equally important |
| Recall | The share of real attacks we caught |
| False alarm rate | The share of normal traffic we wrongly flagged |
| Seed | A number that fixes the random choices, so the experiment can be repeated exactly |
