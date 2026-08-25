# What does a cheaper review panel fail to raise, and does it matter?

**Price: $5.42 · 29 verdicts · `claude-opus-5`, effort `high`, blind, with context packs · 0 error rows.**

**Is this a claim about the instrument, or about an arm?** **About an arm** — specifically about a
*candidate configuration* of the panel arm, not about either of the benchmark's two arms.

**Why it is not in `reports/`.** `reports/` is lane-rendered and keyed by `comparison_id`, and every
`comparison_id` in this project is *our panel vs CodeRabbit*. This measurement has no CodeRabbit side
at all: it compares two configurations of our own reviewer over one corpus. There is no
`comparison_id` for it, and inventing one would put a config experiment where the scoring lane
publishes arm comparisons.

---

## The question

An all-`claude-sonnet-5` panel costs **49.0% less** than the committed panel (5 opus + 1 sonnet) on
the same six pull requests, same `panel_sha`, same day, K=3 both arms — and raises **68% fewer
findings**. Cost is measured; the obvious follow-up is not.

Everything about that comparison was arm-versus-arm, and **the committed panel is not ground truth —
it is the other arm.** So: of the defects the committed panel raised, blocked on, **and an independent
verifier confirmed at high confidence**, which of the ones the sonnet arm never raised are *real*?

A defect never raised is the only kind of loss that no downstream stage can recover. The verifier
never sees it, the gate never sees it, no fix round is spent on it, and nothing records it.

## Answer: 8 real blocking defects lost across 6 PRs, and 7 of the 8 are the security lens

| of 43 distinct control `confirmed-high` blocking defects | | Wilson 95% |
|---|---|---|
| never raised by the variant | **29/43 = 0.674** | [0.525, 0.795] |
| never raised **and real** | 28/43 = 0.651 | [0.502, 0.776] |
| never raised, real, **and worth blocking** | **8/43 = 0.186** | [0.097, 0.326] |

Within the 29 judged:

| | | Wilson 95% |
|---|---|---|
| real | **28/29 = 0.966** | [0.828, 0.994] |
| real **and worth blocking** | **8/29 = 0.276** | [0.147, 0.457] |

**The misses are more blocking-worthy than the panel's average finding.** The pilot's 97 gating
findings measure **0.186** worth-blocking under this same instrument. These misses run at **0.276 —
1.48×**. The cheaper arm is not dropping the panel's noise; it is disproportionately dropping the part
that was load-bearing.

⚠ **That 1.48× is only meaningful because the two figures share an instrument.** Both were adjudicated
by `claude-opus-5`, blind, with context packs — the pilot's under `labels/2026-08-10-pilot-reviewed/`
(see `probes/2026-08-24-adjudicator-instrument` for why the instrument had to be held still). The
pre-pack figure for the same population was 0.134; comparing against *that* would have overstated the
ratio.

**And it is one lens.** Of the 29 missed: `security` 14 · `correctness` 5 · `design-fit` 4 ·
`test-adequacy` 3 · `blast-radius` 2 · `docs` 1. Conversion to real-and-blocking:

| | real-and-blocking / missed | Wilson 95% |
|---|---|---|
| `security` | **7/14 = 0.500** | [0.268, 0.732] |
| everything else combined | **1/15 = 0.067** | [0.012, 0.298] |

## Three ways to lose a blocker, and only one of them is a model problem

The population was frozen with the project's own matcher (`finding-match.mjs` / `groupFindings`),
which separates something a file-level test cannot:

| of 43 distinct control `confirmed-high` blocking defects | n | |
|---|---|---|
| variant raised **nothing matchable** | **29 (67%)** | recall loss — what this probe labels |
| variant raised it, **non-blocking only** | **7 (16%)** | the defect *is* surfaced; the severity→gate mapping drops it |
| variant raised it blocking too | 7 (16%) | no disagreement |

**The middle row is a severity-policy problem, not a model problem**, and pooling it with the first
would attribute this project's own gate diagnosis to a model swap. The clearest case is `pr-471`: the
control's three `confirmed-high` `blast-radius` instances are **two** defects, and the variant raised
one of them — `sheet.ts:3842`, same line, as `minor / correctness`. It was **seen and filed minor**,
not missed.

## Why the population is 29 and not 11

An earlier reading of the same data used a string key plus "did the variant flag the same file
anywhere". It reported 11 misses out of 74 instances. **Both halves of that test were wrong in the
same direction:**

| method | distinct | misses |
|---|---|---|
| string key + same-file generosity | 74 | 11 |
| matcher dedup + same-file generosity | 43 | 9 |
| **matcher dedup + matcher generosity** | **43** | **29** |

- **Dedup 74 → 43.** A `(item, file, summary[:70])` key collapses **none** of the 74 instances —
  models rephrase every leg, so string identity is not defect identity.
- **Test 9 → 29.** **Co-flagging a file is not raising the defect.** In 20 cases the variant touched
  the same file with a *different* finding, which the file-level proxy credits as "raised". The
  generous proxy under-counts misses by **3×**.

## What this does not say

- **Nothing about the variant's own precision.** These are the *control's* defects. Whether the
  sonnet arm's own 113 findings are real is unmeasured, and equal precision per arm would not mean
  quality held: an arm raising a third as many findings at equal precision has lost recall.
- **Nothing about CodeRabbit.** No CodeRabbit findings were read.
- **Nothing about the other 14 defects** in the two non-missed rows. Only the 29 were judged.
- ⚠ **`pr-605` is excluded throughout** — 6 items, not 7. The control arm has zero usable
  observations there: all three legs hit `infra` on a drained credential pool, and a stored `infra`
  item makes a resume skip it forever. Pooling the control's 6 against the variant's 7 would have
  compared a shorter control to a longer variant.
- **One lens carries the result.** `security` is 7 of the 8 lost blockers on n=14. The other five
  lenses combined convert 1 of 15. A different corpus with a different security surface could move
  this a long way.

## What is in `raw/`

| | |
|---|---|
| `tier1-result.json` | the analyser's output — the three-way split, per-lens counts, spend |
| `journals/` | 15 per-item-leg journals (`.jsonl`) and their summaries (`.json`), 29 verdicts total |
| `tier1.log` | the run log, including the per-invocation cost read-back |
| `plans/` | the frozen population: the 43 defects, the 29 differential, and the per-run plans |
| `packs/` | the 6 context packs the adjudicator read — the evidence, so a re-reading sees what this one saw |

**Labels** are not here. They are data and live at
`labels/2026-08-10-pilot-reviewed/findings/<item>/panel/<sha256>.json`, written by this run.
Checked before writing: **0 of 469 arm finding keys collide** with the 97 pilot labels already
published there, so no label was overwritten and no `--relabel` was passed.

## Reproduce

```bash
node reference/model-arm-tier1-population.mjs   # re-freeze the population from the store
node reference/model-arm-tier1-analyse.mjs      # the three-way split and this README's tables
```

Both live in the handoff kit and read the store; neither costs anything.
