# Does the finding-level result depend on the definition of "same defect"?

**Question.** Every adjudication in this project used one rubric — `one-fix-discharges-both`: two
findings are the same only if one fix discharges both. The most plausible alternative is looser and
arguably fairer to the comparison arm, since **CodeRabbit itself reports one finding with the other
affected files listed beneath it**:

> **`same-root-cause`** — one underlying defect, however many places it surfaces. Two findings are the
> same if a human who understands one has understood the other, **even when fixing them takes more
> than one edit.**

Measuring our panel by a unit CodeRabbit does not use makes the arms non-comparable at exactly the
level being compared. So the alternative was tested — **on both arms, under one frozen criterion.**

**Answer: it moves nothing, in either direction.**

| arm | `one-fix-discharges-both` | `same-root-cause` |
|---|---|---|
| cross-replicate | 5 / 150 = 3.3% | **5 / 150 = 3.3%** |
| cross-arm | 21 `same` among 543 existing labels | **1 of 522 `different` labels flipped** = 0.2%, Wilson 95% CI [0.0%, 1.1%] |

Cross-replicate is not merely the same rate. It is **150/150 agreement pair-for-pair, the identical
five pairs, zero flips in either direction.**

| | |
|---|---|
| **verdicts** | **672** — 522 cross-arm + 150 cross-replicate |
| **cost** | **$16.2489** — $12.0222 cross-arm ($0.0230/pair) + $4.2267 cross-replicate ($0.0282/pair) |
| **rubric** | `same-root-cause`, frozen to `raw/rubric-same-root-cause.md`, `sha256:e76af1ef…8de5ea3` |
| **date** | 2026-08-22 |

## The design property, which is the point

**The rubric was frozen and hashed before either arm ran, and applied uniformly to both.** That was
deliberate, because the two metrics were predicted to move in *opposite* directions:

- cross-replicate — reproducibility rises. **This flatters us.**
- cross-arm — more panel findings count as matching CodeRabbit's, so measured distinctiveness of both
  reviewers falls. In a report whose thesis is *where each reviewer wins*, **this costs us.**

Choosing a definition after seeing which way it moves your headline is the pattern that gets a result
dismissed however good the reasoning. Applying one criterion to both arms is the answer to that
objection, and every one of the 672 verdicts records the rubric hash, so the claim is checkable rather
than asserted. **Both halves of the prediction were then falsified: neither number moved.**

## Why nothing moved: the mechanism is absent from the population

`basis` on the cross-replicate arm — the mirror field that says *why* a pair is the same:

```
the 5 `same`:        same-defect-one-site 4 · same-root-cause-multiple-sites 1
the 145 `different`: different-root-cause 80 · unrelated 65
```

**One root cause surfacing at more than one site fires once in 150.** That is the whole mechanism by
which this rubric could have changed anything.

The prior probe's headline was that the answer spanned 3.3% to 55.3% depending on the rubric, on the
assumption that the strict rubric's `related-not-co-extensive` category meant *one cause, several
sites*. Re-adjudicated:

```
the 78 strict `related-not-co-extensive` pairs, under same-root-cause:
    68  different / different-root-cause
    10  different / unrelated
     0  same
```

**Zero.** The instrument keeps *"these overlap in subject but have two causes"* apart from *"one
cause, two sites"*, and 68 of 78 are the former. The span was a conflation of two categories the
adjudicator distinguishes, not a property of the data. The $8 bought the finding that it does not
exist.

## The single cross-arm flip, and why it moves no published figure

`92d498f0f56b` — pr-605, matcher score 0.500, high confidence. `MemNoteStore`'s `undoStack` appends a
full-document text snapshot on every un-batched edit with no cap and no eviction, against CodeRabbit's
"consider capping the snapshot history". One root cause, one site; the strict pass called it
`different`.

Its existing label was **silver**, so gold is untouched by construction. And its CodeRabbit class
`D-e4913d7bc5da14a2` was **already promoted to shared** by silver label `6c8b2076d047`, so a second
`same` on the same class promotes nothing. Verified against this repository at `825bfcf`:

- gold stays **`[7.3%, 20.4%]`** with **`both = 12`** of 165 classes
- silver stays **`[5.4%, 14.2%]`** with `both = 9` of 168

⚠ The prediction on record was that the bands would **tighten**, since resolving any `maybe` removes
uncertainty. **They did not move at all** — the prediction assumed movement, and there was none.

## `insufficient-basis`: 0 of 672, and 0 of 1,053

Both prompts carried an explicit decision procedure naming the checkable circumstance in which the
third verdict is **required** — either finding failing to identify a mechanism. **It was used zero
times.** Across every adjudication pass in this project that is **0 uses in 1,053 verdicts.**

This is no longer a curiosity. A decision procedure was added specifically to make the verdict
reachable and did not change the count by one. The honest statement: **a three-verdict schema is being
run as two.** Either reduce it to two, or investigate the unreachability as a property of the model
rather than of the prompt.

## Class level, beside pair level, never multiplied

Because the five cross-replicate matches are the *identical* five, the class-level reading is
unchanged from the strict pass: 4 confirmed class merges, two of which create a new all-three class →
**recurrence 56/245 = 22.9% → 58/241 = 24.1%.** Still no upper bound: 2,802 class pairs over 245
classes is a mean degree of 22.9, so merges chain and a per-pair rate multiplied into the population
yields a negative class count.

Cross-arm: 1 flip, **0 class state changes, both bands unmoved.**

## What this does NOT establish

- 🔴 **`basis` is missing for all 522 cross-arm verdicts — a real data loss.** The schema required it
  and the model returned it; the journal row enumerated three keys by name and discarded it. So the
  split of the 521 `different` verdicts into `unrelated` versus `different-root-cause` **is
  unrecoverable** without re-buying 522 pairs at ~$12. **Not recommended:** the 0.2% headline does not
  use it, and the cross-replicate arm already shows that split running 80/65. Recorded so the gap is
  visible rather than absent.
- **Any rubric other than these two.** Two definitions of "same defect" now agree. A third could
  differ.
- **150 is small.** Wilson 95% CI on the cross-replicate rate is [1.4%, 7.6%]. The load-bearing claim
  is not the rate but the **150/150 pair-for-pair agreement**, which is a stronger statement than two
  rates coinciding.
- **Complete linkage on multi-link class pairs.** 1,575 class pairs still have zero full coverage.
- ⚠ **The cost estimate was 2× low** — $16.2489 against a planned ~$8.06. `--max-calls` bounds calls,
  not dollars (668 calls against 740 caps), so the estimate was the only guard. Measured per-pair cost
  varies 2.5× across arm × rubric and the cheapest cell was applied to all four:

  | arm / rubric | $/pair | output tokens/pair |
  |---|---|---|
  | cross-arm / `one-fix-discharges-both` | **$0.0120** ← planned with this | 311 |
  | cross-arm / `same-root-cause` | $0.0230 | 929 |
  | cross-replicate / `one-fix-discharges-both` | $0.0301 | 1286 |
  | cross-replicate / `same-root-cause` | $0.0282 | 1149 |

## Invariant 3 — instrument or arm?

**This is a claim about the INSTRUMENT** — whether the adjudication criterion, not the reviewers,
determines the finding-level result — so `reports/` is not where it belongs. The arm figures it
quotes (`[7.3%, 20.4%]`, `both = 12`, `[5.4%, 14.2%]`) are the lane's own published numbers, shown
unmoved rather than restated as new results.

🔴 **No label was written, and there is nowhere yet to put one.** `pairLabelKey` hashes
`file|line|summary` on both sides and nothing else — deliberately rubric-blind, so *a human's verdict
is not lost because the matcher was refactored*. A `same-root-cause` verdict therefore produces the
**identical filename** as its `one-fix-discharges-both` verdict, and writing it would overwrite the
existing label, including 31 gold ones. The storage fix, if this rubric is ever adopted, is a
rubric-scoped path (`labels/<corpus>/pairs/<rubric>/`) — **not** a key change. A later decision.

## `raw/`

| file | what it is |
|---|---|
| `raw/rubric-same-root-cause.md` | the criterion, frozen before the run, `sha256:e76af1eff946d8e3a5a2a73abd42957178199de49dc2ee5b54e12acc18de5ea3`. Outcome-free, covering both arms |
| `raw/work-list-cross-arm.json` | the frozen enumeration: 522 keys sorted lexically, shard = index % 4 → 131/131/130/130. Also records the provenance — 543 labels on disk = 491 silver-`different` + 31 gold-`different` + 21 `same` carried over, plus the three keys that moved across #801 |
| `raw/work-list-cross-replicate.json` | the same, 150 keys → 38/38/37/37 |
| `raw/journal-cross-arm-shard{0..3}.jsonl` | 4 journals, 526 rows → **522 verdicts** + 4 failures at $0 |
| `raw/journal-cross-replicate-shard{0..3}.jsonl` | 4 journals, 150 rows → **150 verdicts**, no failures |

Every journal row carries `rubric`, `rubric_sha256` and `prompt_sha256`. Reconciled across all eight:
672 verdicts, zero duplicate pair keys, every assigned pair answered, one rubric hash, and **exactly
two prompt hashes — one framing per arm sharing one criterion** (`0eec013ebf3ba928…` cross-arm,
`3bb33e2f3855de4b…` cross-replicate), each checked against the prompt the shared rubric module
composes for that arm rather than merely counted.

**Not copied:** the eight per-shard run artefacts. They restate the enumeration already pinned in the
work lists and derive their roll-ups from the journals, so they are re-derivable at $0 — and unlike
the journals they carry a local absolute path in `coderabbit_source`. The frozen CodeRabbit-arm read
the cross-arm shards consumed is
[`../2026-08-21-cross-arm-tail/raw/cr-arm-k2-cache.json`](../2026-08-21-cross-arm-tail/raw/cr-arm-k2-cache.json),
`sha256:d4fa58be…3974a3`; the 543 labels they read are at `labels/2026-08-10-pilot-reviewed/pairs/`,
committed at `825bfcf`.
