# Is the panel's "unreliable at finding level" result real, or a matching artefact?

> ## ⚠ SUPERSEDED IN PART — the sensitivity was tested and it is zero
>
> This probe's report closed with: *"The probe's answer spans 3.3% to 55.3% purely on the definition
> of 'same defect.'"* **On 2026-08-22 that sensitivity was measured and it does not exist.** The same
> 150 pairs were re-adjudicated under a frozen `same-root-cause` rubric and the 78 pairs whose
> `basis` was `related-not-co-extensive` came back **68 `different-root-cause`, 10 `unrelated`, 0
> `same`.** Pair-for-pair agreement between the two rubrics was **150/150.**
>
> Be fair to the original: it already labelled 55.3% *"a sensitivity, not a measurement."* The
> correction is not *"it was wrong"* but **"it was tested and it is zero."** The 3.3% below stands
> unchanged and is now known to be rubric-independent.
>
> **Superseded by:** [`probes/2026-08-22-same-root-cause/`](../2026-08-22-same-root-cause/).

---

**Question.** `reliability.mjs` stamps every figure it emits `"bound": "lower"`, and its own header
says why: two findings from different runs are *cross-source* to `gateFor`, so they take the L2 gate,
which answers `maybe` for a location tie that misses the token bar — and a `maybe` never merges, so
every unmerged same-defect pair splits one class in two, inflating the union and deflating the ratio.
**5,565 such pairs existed and nobody had ever read one.** If many are true merges, the pilot's
headline — *reliable at the decision level, unreliable at the finding level* — is partly an artefact
of matching rather than a property of the panel.

**Answer: the artefact is small, and the direction is up.**

| level | result |
|---|---|
| **pair** | **5 / 150 = 3.3%** · Wilson 95% CI [1.4%, 7.6%] |
| **class** | **4 confirmed merges → recurrence 56/245 = 22.9% → 58/241 = 24.1%**, measured |

| | |
|---|---|
| **corpus** | `2026-08-10-pilot-reviewed`, the three pilot replicates, K=3, linkage `complete` |
| **verdicts** | **150** (5 `same`, 145 `different`) |
| **cost** | **$4.5144** · **$0.0301/pair** |
| **date** | 2026-08-21 |

## 🔴 The upper bound is deliberately absent

A per-pair rate cannot be scaled into this population and it is important to say so rather than
publish the number that results. The 5,565 links span **2,802 distinct class pairs over 245
classes — a mean degree of 22.9**, so merges chain: resolving one changes which others are still
distinct. Naive scaling gives 11.8% × 2,802 ≈ 331 merges against 245 classes, i.e. **a negative class
count.**

So: **24.1% is a measured lower bound. There is no upper bound in this probe**, and the reason is
structural rather than a sample-size problem. Establishing one needs full coverage of class pairs
with more than one link, which is the next purchase.

## Strata, `n` on every figure

| stratum | matched / n |
|---|---|
| same file | **5 / 53** |
| cross file | **0 / 97** |
| same lens | 2 / 39 |
| cross lens | 3 / 111 |
| both findings `nit` | **2 / 5** |
| exactly one `nit` | **0 / 31** |
| neither `nit` | 3 / 114 |
| matcher score < 0.50 | **0 / 91** |
| matcher score ≥ 0.50 | 5 / 59 |

**Every match is same-file, and cross-file is 74% of the population.** All five sit at 0.500 or
above; the lowest is exactly 0.500.

The `nit` contrast is the sharpest signal here and both halves belong in any quote of it: 2 of 5
when *both* findings are nits, **0 of 31** when only one is. `nit` is also the pilot's largest apparent
unreliability (3/26 = 11.5% pairwise class jaccard), which is why it is worth a probe of its own.

**The same-lens / cross-lens split is a property of the metric, not noise.** Within a run, two
findings must share lens *and* file or the matcher early-returns `no`. Across runs they take L2, which
does not require the same lens — so a cross-lens merge is possible cross-run and impossible
within-run. Smoothing the two strata together hides that.

## The threshold in play is 0.3, not 0.22

Cross-run pairs are cross-*source* but **same-arm**, and `finding-match.mjs:303` reads
`crossSource && !sameArm ? CROSS_ARM_SIMILARITY : DEFAULT_SIMILARITY`. So these pairs are gated at
`DEFAULT_SIMILARITY = 0.3`. `CROSS_ARM_SIMILARITY = 0.22` is not involved, and the cross-arm tail
result does not transfer here. Verified against `wafflebase/wafflebase` `main` at
`148bfcb4e171de4adbb575ec585f415b09515aee`.

## Population, reconciled before spending

| | mine | published |
|---|---|---|
| `maybe_cross_run` | **5,565** | **5,565** |
| `maybe_within_run` | 74 | 74 |
| `match_held_apart` | 264 | 264 |

Against `recurrence.unmerged` in
`scores/by-config/…__2026-08-10-pilot-reviewed/reliability-v1.json`. ⚠ **That file states the
unmerged census twice and the two copies disagree** — `jaccard.unmerged_total` reads 5565 / **148** /
**244**. This is scope, not a defect: `recurrence` groups all three runs in one pass while `jaccard`
groups pairwise, so a within-run pair is counted in both groupings containing its run (74 × 2 = 148)
and complete linkage resolves differently over three runs than over two. A **cross-run** pair appears
in exactly one pairwise grouping either way, which is why 5,565 agrees in both.

**Two neighbouring populations, named and not sampled:** `match_held_apart: 264` (pairs the matcher
called `match` that complete linkage declined) and `maybe_within_run: 74` (the over-emission
question). Different populations; not interchangeable with `maybe`.

## What this does NOT establish

- **Any upper bound.** See above. 34 of 2,802 class pairs were covered — 1.2%.
- **Complete linkage on multi-link class pairs.** The four confirmed merges are each on a class pair
  whose *only* link was judged, so complete linkage is satisfied rather than assumed. The fifth
  matched pair has 5 sibling links, 1 judged, and remains undetermined. **1,575 class pairs still have
  zero full coverage.**
- **That the reliability figures were right.** ⚠ This is the trap in reading a *low* rate, and it
  inverts the usual warning. The adjudication rubric resolved *"related but not co-extensive"* toward
  `different` — cross-arm that direction over-counts distinct findings (unflattering, safe to publish);
  **cross-replicate the same bias suppresses merges.** So *"the reliability numbers were basically
  right"* is the artefact-prone reading here, not the conservative one. **78 of the 145 `different`
  verdicts turned on co-extensiveness rather than on the two findings being unrelated** — and it is
  exactly those 78 that the `same-root-cause` probe went on to re-test, finding 0 flips.

## A defect found in this probe's own instrument

`same_lens` was computed as `undefined === undefined`, because finding records carry no top-level
`lens` — it lives at `record.panel.lens`. The comparison returned `true` for every pair and reported
a spurious 5565/5565 same-lens population. Found and fixed before any figure was published; the
truth is **4,339 of 5,565 cross-run maybes (78%) are cross-lens.**

## Invariant 3 — instrument or arm?

**This is a claim about the INSTRUMENT** — how much of `reliability.mjs`'s lower bound is matching
rather than panel behaviour — so it does not belong in `reports/`. It is also why nothing here was
promoted: *resolving a `maybe` is adjudication, not scoring.* No scorer was touched, no `maybe`
promoted, and **no pair label written** — `pair-labels.mjs` is cross-arm by construction, keyed by
`pairLabelKey(panelFinding, coderabbitFinding)`, so a panel-vs-panel verdict would be schema-valid
and semantically wrong.

## `raw/`

| file | what it is |
|---|---|
| `raw/journal.jsonl` | 156 rows — one per model call. **150 verdicts** plus 6 failures charged $0. Each row carries the pair's item, both `run_id`s, both lenses, both severities, both files, the class ids, the withheld matcher score/verdict, and the verdict with its `basis` — every stratum in the table above is recomputable from this file alone |
| `raw/run-artefact.json` | the run receipt: the population reconciliation against the published census, the sampling rule as executed, the strata roll-up, and `writes_pair_labels: false` |

**Not copied — the 5,565-pair enumeration (7.7 MB).** It is *derived*, not an input this store lacks:
`matchFindings` is free and the enumeration is computed from the three replicates' finding records,
which this repository already holds under `runs/`. Copying it would duplicate 7.7 MB of re-derivable
data and would breach *"never copy something the store already holds."* The reconciliation figures
that make it checkable are in the table above and in `run-artefact.json`; the sample's membership is
pinned per-pair in the journal.
