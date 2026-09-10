# A 427k-parameter interpreter for decision tables

> 2026-09-10 / Eris @ Three Hearts Space
> Written in English; this file is the record of truth for this note.

## In one line

A 427k-parameter network was trained on machine-generated decision tables only, using a
vocabulary of 15 symbols that contains **no domain words at all**, and it resolved nine
held-out hand-written tables (1,749 rows) at 100.0% across three seeds. The interesting
part is not the number — it is which measurements had to be repaired before the number
meant anything, and which of them still do not.

---

## The task

A **decision table** has named axes, each with a finite set of values, and a list of
clauses. A clause has a guard (constraints on some axes), a rank, and an outcome. A
**face** is one complete assignment of values to every axis. Resolving a face means
finding which outcome the table gives it.

The semantics being modelled:

```
1. a clause fires        ⇔ every axis is MATCH or unconstrained          (AND)
2. if any firing clause carries "impossible", the answer is impossible    (absolute priority)
3. otherwise the winner is max by (rank, position in the outcome list)
4. if nothing fires, the face is a hole
```

An exact engine already resolves this by enumeration. The question is whether a network
can learn **the procedure** rather than any particular table — that is, whether it can be
handed a table it has never seen, in its input, and read it.

The nearest familiar shape: this is closer to writing an interpreter than to writing a
program. The rules live in the input, not in the weights.

## Encoding: fifteen symbols, none of them domain words

Each face is presented as a sequence of fixed-width clause blocks. Every clause becomes
eight slots:

| slot | contents |
|---|---|
| 0 | `CLS` (live) or `PAD` (dead clause, not computed) |
| 1 | `R{k}` — the clause's strength **as a rank within this table** |
| 2 | `O{i}` (outcome index) or `IMP` (the impossible outcome) |
| 3–7 | per axis: `MATCH` / `NOMATCH` / `ANY` / `AXPAD` |

The complete vocabulary is:

```
PAD, CLS, IMP, ANY, MATCH, NOMATCH, AXPAD, R0..R3, O0..O3
```

No axis name, no value name, no outcome name is ever tokenised. A table about deployment
policy and a table about numbers reduce to the same alphabet. This is what makes the
question well-posed: a model that scores on unseen tables cannot have memorised their
contents, because their contents never entered the vocabulary.

Slot 1 is worth isolating. An earlier version wrote the raw rank clamped to a maximum.
That was wrong twice: distinct ranks silently collapsed into one symbol, and the *same*
relative strength encoded differently in different tables (a table whose ranks start at 1
gave its weakest clause a different symbol than a table whose ranks start at 0). Since the
semantics only ever *compares* strengths, the absolute value carries no information —
only the order does. Mapping rank to its position among the table's distinct ranks fixed
both faults.

Frames are declared and enforced: at most 5 axes, 60 clauses, 4 ranks, 4 outcomes. A table
outside any frame raises rather than being truncated (see *Silent truncation*, below).

## Architecture

```
clause block (8 slots)
  → per-slot embedding table (15 × 8 entries, dim 40)
  → concatenate  → 320
  → Linear(320,384) → GELU → Linear(384,384) → GELU → LayerNorm
clause representation (384)          dead clauses are not computed
  → fold across clauses
  → Linear(384,384) → GELU → Linear(384,5)
outcome (5 classes)
```

427k parameters. **No attention anywhere.**

Attention was removed deliberately. The eight slots of a clause block have roles fixed by
position — which slot means "rank" never varies — so there is nothing to learn about where
to look. Concatenating the slot embeddings and passing them through an MLP expresses the
same function for roughly a third of the compute. Attention earns its cost when the
relevant position depends on the input; here it does not.

The consequence is that clauses cannot interact except through the fold. Compute is
therefore **linear** in the clause count, not quadratic.

Three folds were compared. `max` takes the elementwise maximum of the clause
representations. `pick` emits one scalar strength per clause and selects a single clause by
argmax. `soft` uses a softmax over those strengths with a learned temperature.

`max` is the current default and reaches 100%, but it **cannot represent the semantics as
stated**: 384 dimensions are maximised independently, so the pooled vector can be assembled
from clauses that do not fire. Instrumenting the pre-fold representations on an earlier,
weaker checkpoint showed exactly that — on one table's eight errors, 45% of the pooled
dimensions came from non-firing clauses. That an architecture which provably cannot express
the rule nonetheless scores 100% is evidence about the difficulty of the current test set,
not about the architecture.

`pick` and `soft` were indistinguishable from `max` in every comparison run. That is also a
statement about difficulty rather than about folds.

## Training

- Training data: 540 machine-generated tables, 53,883 faces.
- Held out for stopping: 60 machine-generated tables, 6,199 faces.
- Test: 9 hand-written tables, 1,749 faces, **never shown during training**.
- AdamW (lr 3e-3, weight decay 0.01), OneCycle, gradient clipping 1.0, batch 512, 30 epochs.
- Cross-entropy with inverse-frequency class weights, to stop the model from borrowing the
  training set's outcome prior.
- One epoch takes about 10 seconds on four CPU cores.

Stopping uses the **loss** on the held-out machine tables, not their accuracy. Accuracy
there saturates at 100% by epoch 5 and never moves again, so it cannot point at a maximum;
an accuracy-based stop selected epoch 4, worth 79.3% on the real tables, where a
hand-chosen epoch 10 was worth 98.6%. Loss keeps moving after accuracy saturates.

Each run records the arguments and a hash of the exact table set on its first line. Two
runs with different hashes are not compared. This was added after a run appeared to change
behaviour when the real difference was a regenerated corpus.

## Results

Nine hand-written tables, three seeds, identical:

| table | faces | accuracy | majority-class baseline |
|---|---|---|---|
| A | 72 | 100.0% | 48.6% |
| B | 81 | 100.0% | 60.5% |
| C | 144 | 100.0% | 52.8% |
| D | 780 | 100.0% | 80.5% |
| E | 48 | 100.0% | 41.7% |
| F | 240 | 100.0% | 63.3% |
| G | 240 | 100.0% | 63.3% |
| H | 48 | 100.0% | 50.0% |
| I | 144 | 100.0% | 62.5% |
| **total** | **1,749** | **100.0%** | **58.1%** |

With an abstention threshold of 0.9 the model answers on 100.0% of faces.

Three floors were computed to check the instrument itself:

| set | faces | accuracy | majority | chance floor Σq² |
|---|---|---|---|---|
| hand-written tables | 1,749 | 100.0% | 68.7% | 53.5% |
| held-out machine tables | 13,400 | 100.0% | 60.6% | 49.1% |
| label-shuffled tables (negative control) | 15,149 | 49.6% | 61.6% | 49.6% |

The second row is the load-bearing one: 13,400 faces of randomly generated tables that no
one has inspected, resolved exactly. The third row is the instrument check — a model that
scored above Σq² on tables whose labels had been shuffled would indicate a measurement
fault, and it does not (difference −0.1%).

## What had to be repaired first

Five of these were mine, and each of them produced a number that looked like a result.

**A mean where the semantics is a max.** The first version folded clause representations by
averaging. The rule being learned is "the strongest firing clause wins" — an argmax.
Averaging actively destroys it. That version scored 31.0% against a 59.1% majority
baseline: worse than answering the most common class. The conclusion available at the time
("the procedure cannot be learned") was not supported; the fold was wrong.

**Length extrapolation, misdiagnosed as failure to transfer.** The second version trained
on tables of 5–11 clauses and was tested on one of 30. It scored below that table's
majority baseline. Regenerating the training corpus across 3–60 clauses removed the
deficit. Nothing about learning capacity had been measured.

**A guard form that was invisible to the encoder.** Guards may be expressed as a negation —
"any value except these". The encoder tested membership with an operation that, applied to
that form, inspected the wrong structure and returned NOMATCH unconditionally. Four such
clauses existed in one table, and they were the deciding clause for 304 of its 780 faces
(39.0%). The reachable ceiling was therefore 61.0%. An earlier run had scored **exactly
61.0%** and been recorded as a partial result. The model had been scoring full marks
against a broken input.

**Silent truncation in four places.** Tables exceeding a frame were quietly clamped: ranks
past the fourth merged, outcomes past the fourth merged, axes past the fifth dropped,
clauses past the sixtieth sampled away. Each produced training pairs whose input no longer
determined the label — the guard that decided the answer was no longer visible, but the
answer stayed. All four now raise and name the table.

**A control that was not a control.** A structure-free table generated from the digits of π
was used as a negative control. It has 216 clauses against a 60-clause frame, so 156 of its
clauses were being sampled away before the model saw it. Its score had been read as
"structure-free tables still beat majority, so the measurement is sound". That reading was
about truncation, not structure. The control seat is currently empty.

**A safety metric that could be satisfied by doing nothing.** One diagnostic counts faces
where the model's error moves toward the less restrictive outcome. That count goes to zero
for any model that always answers the dominant class — and one run did exactly that,
reporting zero such faces while scoring 41.0% on a table with an 80.5% majority. The
diagnostic now refuses to be read without accuracy and baseline printed beside it.

**A floor asserted instead of computed.** The negative control's expected score was written
down by eye as 0.45 and the observed 49.6% was flagged as instrument failure. The correct
floor for that control is Σ_c q_c², which is 49.6%. The instrument was healthy; the
expectation was not calculated.

## Error structure, before the corpus was fixed

The repairs above raised the hand-written tables from a mean of about 90% to 100%. The
error structure measured just before that is more informative than the final number,
because it shows how this class of model fails.

Aggregate accuracy hides the kind of error. On one 240-face table scoring 56.7% overall,
splitting by the kind of face gives:

- faces where some clause speaks and gives a definite outcome: **97.7%** (88 faces)
- faces whose true class is the impossible/no-rule class: **47.4%** (152 faces)

Of 82 errors, **80 (98%) were in one direction**: the true class was the restrictive one,
and the model answered a definite outcome instead.

The mechanism was recoverable. Extracting the effective rule the model had implemented and
rewriting it into a canonical form showed clauses of the form "one axis matches ⇒ definite
outcome" standing in for a multi-axis guard whose other axes had been ignored. The model
was **dropping an axis**, and the faces it thereby mis-resolved were, disproportionately,
the ones the table restricts.

Two further diagnostics were built around this:

- **Complexity as a label-free error signal.** Both the model's effective rule and the
  table's true rule were rewritten into the same canonical form and their clause counts
  compared. Tables where the model's rule is *more* complex than the true rule are the ones
  it gets wrong; tables where it is simpler are the ones it gets right. This is computable
  without labels.
- **Cross-seed intersection.** Faces flagged as risky by independent seeds were intersected.
  A high intersection ratio indicates systematic error; a low one indicates variance. This
  separated a persistent 12-face cluster from a 6-face cluster that moved between runs.

After the training corpus was regenerated to match the shape distribution of hand-written
tables, the flagged-face count went from 18 to 0 across all seeds.

## What these numbers do not show

**The generator's shape was fitted to the test set.** The distribution used to sample
training tables — axis counts, values per axis, clause counts, proportion of impossible
outcomes, rank distribution — was chosen by measuring those statistics on the nine
hand-written tables. The contents were never seen, but the shape was. The defensible claim
is therefore narrower than "never saw the real tables": it is **"given tables of matching
shape, the contents need not be seen."**

That fit mattered. Before it, sampling axis counts uniformly across 2–6 produced a corpus
that was 87% two- and three-axis tables, because the number of values per axis was drawn
independently and large draws hit the total-face cap before the axis count could grow. Two
dials that were believed independent were coupled, and one of them was not actually varying.

**The benchmark is saturated.** One seed reaches 100.0% on every hand-written table at
**epoch 2** and stays flat for the remaining 28 epochs. Nothing about capacity, fold choice,
or training length is currently measurable, because everything scores the same. Finding
where the model breaks is a prerequisite to any further claim about it.

**Extrapolation is untested.** No run has been made outside the declared frames, and no run
has held out an interior region of the shape space (for example, training on 2, 3 and 5
axes and testing on 4). Interior holes are the stronger test, since they cannot be
dismissed as insufficient capacity.

**Half the procedure is precomputed.** Whether a guard matches a given axis value is
computed by the encoder and handed to the model as `MATCH`/`NOMATCH`/`ANY`. What the model
learns is the AND over slots, the priority of the impossible class, and the argmax over
(rank, outcome position). Pattern matching itself is not learned.

**The learned circuit has not been read out.** In the `pick` fold, one scalar per clause is
the only channel between clauses, and the semantics predicts what that scalar must encode:
a lexicographic key over (rank, outcome position), with a large negative offset when any
axis is NOMATCH. Regressing the learned scalar against that form would convert "100%
accurate" into a specific claim about which circuit was learned. It has not been run.

## Reproduction

The model, the table generator, and the diagnostics are a few hundred lines of PyTorch and
plain Python; a full three-seed sweep takes about fifteen minutes on four CPU cores. The
elements needed to reconstruct it are given above: the fifteen-symbol vocabulary, the
eight-slot clause block, the per-clause MLP with a fold across clauses, inverse-frequency
class weights, and early stopping on held-out loss rather than held-out accuracy.
