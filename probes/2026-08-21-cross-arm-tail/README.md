# Are cross-arm matches hiding below the matcher's threshold?

**Question.** `finding-match.mjs` scores every panel-vs-CodeRabbit pair and declines to merge below a
threshold. The pairs it scores lowest had never been read by anyone. If real merges are down there,
every published overlap figure understates how much the two reviewers agree.

**Answer: no.** Of the **167 adjudicated pairs scoring below 0.500, zero are matches.** The lowest
`same` verdict sits at exactly **0.500**; the highest-scoring pair below the threshold is **0.4667**,
and it is a non-match.

| | |
|---|---|
| **replicate** | `pilot-01__k2`, corpus `2026-08-10-pilot-reviewed` |
| **verdicts** | **231** (15 `same`, 216 `different`) |
| **cost** | **$2.7739** · **$0.0120/pair** |
| **adjudicator** | `claude-opus-5`, effort `high`, no repository access, one session per pair, matcher score and verdict withheld |
| **date** | 2026-08-21 |

## The tail is a point mass, not a continuum

The enumerated population is 692 within-item cross-arm pairs — 147 panel findings × 30 CodeRabbit
findings over 7 items. Its score distribution:

```
   266  0.00           no location tie and no shared anchor
     0  (0.00, 0.30)   literally empty
   114  [0.30, 0.32)   the audited floor itself
   189  [0.32, 0.50)
    99  [0.50, 0.6875]
    24  (0.6875, 1.00]
```

Nothing lies strictly between 0 and 0.30: the L2 cross-source path does not compute a low score, it
declines to score. So "the low tail" is two disjoint populations — 266 pairs at exactly 0, and 114 at
the floor — and a single threshold conflates them. None of the 266 has a missing `file`; the zeros
are genuine no-overlap, not absent data.

## Adjudicated, by band

| band | population | adjudicated | `same` | rate |
|---|---|---|---|---|
| exactly 0.00 | 266 | 146 | **0** | 0% |
| [0.30, 0.32) | 114 | 2 | 0 | 0% |
| [0.32, 0.50) | 189 | 19 | 0 | 0% |
| [0.50, 0.6875] | 99 | 40 | 4 | 10.0% |
| (0.6875, 1.00] | 24 | 24 | 11 | 45.8% |

The 15 `same` scores in order: `0.500 0.586 0.663 0.667 0.700 0.750 0.818 0.821 0.833 0.875 0.900
1.000 ×4`.

### ⚠ The separation is one-directional. Read it precisely.

"No match below 0.500" is what was measured. It is **not** the claim that the score separates matches
from non-matches: of the 64 adjudicated pairs at 0.500 or above, **49 are non-matches**, two of them
scoring 0.9 or higher and one scoring exactly 1.000. A high score is not evidence of a merge. The
result is a **floor** — below 0.500 the region is empty — and nothing more.

## Also established

- **The published overlap figures count CLASSES, not findings.** `resolveClasses` iterates classes
  whose claim is `coderabbit-only`; `overlapOf` counts rows with `claim === "both"`. A finding-level
  count and the published number are different units and must never be compared directly.
- **Finding → class fan-in is 1:1 on this data**: the 30 CodeRabbit findings sit in 30 distinct
  classes, none holding more than one.
- **All 30 CodeRabbit findings are now decided** — 13 have an adjudicated partner in the panel arm, 17
  provably have none (every pair decided, none `same`). Before this run, 0 of 30 were finished in the
  sense `pair-labels.mjs` requires.
- **Deriving from the real scorer reproduces the published gold figures exactly**: `[7.3%, 20.4%]`
  with `both = 12` of 165 classes. That agreement is what makes the derivation above trustworthy.
- **Tier is not poolable.** The gold band above is gold. The silver band is a separate, tighter
  `[5.4%, 14.2%]` over a larger and lower-confidence label set. Never add them.
- **5 of the 43 applied gold labels matched only via `pair_key_at_801`** — indexing both key vintages
  is load-bearing on this data, not defensive coding.

## What this does NOT establish

- **One replicate.** `pilot-01__k2` only. Nothing here says the floor holds on k1 or k3.
- **Nothing about the threshold's setting.** ⚠ Do **not** read this as "0.22 is 2.3× too low." The
  docblock at `finding-match.mjs:205–227` (constant at `:227`) shows `CROSS_ARM_SIMILARITY = 0.22` is
  a **Dice-scale translation** of `DEFAULT_SIMILARITY = 0.3` — measured factors 0.67 and 0.61 for the
  two callers, so 0.30 → 0.20 and 0.30 → 0.18 — not an arbitrarily low bar. Comparing 0.22 to 0.500
  compares two different quantities.
- 🔴 **An unresolved tension, recorded rather than settled.** That same docblock names a **0.231
  cliff, where the title-shaped caller starts losing true matches.** This probe found no true match
  below 0.500. Both cannot be describing the same population the same way. **Do not resolve it from
  this probe** — it needs the positive set that calibration used, which was six pairs selected by the
  metric being replaced.
- **A band movement.** 154 of the 231 verdicts are on pairs outside the `maybe` queue, including all
  146 score-0 pairs, which were never promotion candidates. Of the 186 labels this probe wrote, 34
  reached the queue and **zero class states changed** — re-derived from the real `resolveClasses`
  against this repository at `825bfcf`. The `[5.4%, 14.2%]` silver band and its moved
  ceiling date from 2026-08-13 and belong to the earlier 312 labels. The baseline for attributing
  movement is the **published** state, never `resolveClasses`'s unadjudicated `band.before` row.
- **Symmetric confidence.** The gold set backing the grade has one uncontaminated positive, so this
  constrains **false** merges far better than missed ones.

## Invariant 3 — instrument or arm?

**This is a claim about the INSTRUMENT** — about what `finding-match.mjs`'s low-score region
contains — so `reports/` is not where it belongs. The arm figures it quotes (`[7.3%, 20.4%]`,
`both = 12`) are not new results: they are the lane's own published numbers, reproduced here only to
prove the derivation was reading the right unit.

## `raw/`

| file | what it is |
|---|---|
| `raw/journal.jsonl` | 235 rows — one per model call. **231 verdicts**, plus 4 turn-ceiling failures on 3 pairs, all retried and resolved, charged $0. Carries the withheld matcher score/verdict alongside each verdict, which is what makes the band table checkable |
| `raw/run-artefact.json` | the run receipt: the full 692-row enumeration with per-pair scores and bands, `band_census`, `local_verdict_census` (match 8 · maybe 418 · no 266), per-item counts, and the label-join statistics. The **denominators** live here — the journal alone cannot say "146 of 266" |
| `raw/cr-arm-k2-cache.json` | the frozen input. The CodeRabbit arm is not in this store, so it was read live once and frozen; `sha256:d4fa58be…3974a3`, 30 in-window findings, `read_at_utc 2026-08-21T03:05:30.549Z`. Without it, two passes enumerate two different populations and no count is comparable |

**Not copied:** the pair-level tail analysis, the class mapping and the baseline check. All three are
derived from the above by scripts in the working kit and are re-derivable at $0. The **186 pair labels
this probe produced are already in this repository** at `labels/2026-08-10-pilot-reviewed/pairs/`,
committed at `825bfcf`.

## 🔴 One redaction, declared

`run-artefact.json`'s `.coderabbit_source` field named the cache by absolute local path. It now reads:

```
cache cr-arm-k2-cache-2026-08-21.json (read_at_utc 2026-08-21T03:05:30.549Z)
```

The filename and the `read_at_utc` are preserved; only the directory prefix is gone. Verified as the
single difference from the kit original — every other byte, including all 692 rows and all 231
results, is identical. No conflict with `raw/` immutability: nobody outside holds a competing copy of
this file, so there is no byte-identity to preserve against.
