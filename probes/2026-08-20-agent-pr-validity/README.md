# Are the panel's findings real?

**Question.** The review panel raises findings on agent-authored pull requests and gates merges on
them. Nobody had ever checked, blind, whether those findings describe real defects.

**Answer: yes — and severity is the problem instead.**

| | |
|---|---|
| **precision** (is it a real defect) | **0.960** — 96 of 100 · Wilson 95% CI [0.902, 0.984] |
| **blocking precision** (would an independent blind reader also block a merge) | **0.210** — 21 of 100 |

The two differ by a factor of ~4.6 and **both belong in any quote of this probe.** The panel's
detection is good enough to trust; its severity is not.

| | |
|---|---|
| **corpus** | `2026-08-20-agent-prs` — 16 fully agent-authored PRs (694, 695, 734, 737, 757, 770, 786, 809, 810, 862, 863, 864, 867, 873, 876, 899) |
| **run** | `live-2026-08-20__agent-prs-allrounds`, arm `panel` |
| **labels** | 100, tier `silver` |
| **cost** | **$129.6160** over **217 charged adjudication calls** across three passes |
| **date** | 2026-08-20 (labels), 2026-08-21 (dataset export) |

## Severity is the result

Adjudicated severity, against a population that is **critical/major by construction**:

```
minor 76 · major 23 · nit 1
```

Panel severity → adjudicated severity:

```
major -> minor      75
major -> major      21
critical -> major    2
major -> nit         1
critical -> minor    1
```

Every one of the 100 findings gated a pull request. Three quarters of them read as `minor` to a blind
reader with the same evidence.

## Per lens, `n` on every row

| lens | n | real | precision | blocking | blocking precision |
|---|---|---|---|---|---|
| `design-fit` | 25 | 25 | 1.000 | 1 | **0.040** |
| `test-adequacy` | 18 | 18 | 1.000 | 0 | **0.000** |
| `docs` | 1 | 1 | 1.000 | 0 | 0.000 |
| `security` | 19 | 18 | 0.947 | 10 | **0.526** |
| `correctness` | 22 | 21 | 0.955 | 5 | 0.227 |
| `blast-radius` | 15 | 13 | 0.867 | 5 | 0.333 |

`design-fit` and `test-adequacy` together contributed **43 real findings and 1 blocking one.**
`security` is the only lens above one in three.

By PR state: merged 43 (precision 0.977, blocking 0.256) · draft 57 (0.947, 0.175).

## Delivery, and the confound found afterwards

> *Shared with a teammate 2026-08-21: 0.960 on n=100. The unconfounded single-instrument reading is
> 0.952 on n=84 — 84 labels are `claude-fable-5`, 16 `claude-opus-5` after a mid-run quota
> exhaustion, and all four false positives are Fable's. Known after the hand-off.*

The 16 `claude-opus-5` labels are pr-786 and pr-695 only, so the split is confounded with item
identity and cannot be read as a model comparison. `annotators` on each row carries it per label.

## 🔴 One noteworthy consequence, worded carefully

> **This is evidence against upgrading the verifier's model.** 96 of 100 verifier-surviving findings
> are real, so existence-checking is already working and a stronger model has little headroom. ⚠ State
> it that way and **not** as *"our Opus-5 verifier scores as well as Fable"*: Fable was the
> **adjudicator**, not a competing verifier; the 0.960 is the precision of the population that
> **already passed** the verifier, so with no counterfactual it cannot be attributed to the verifier
> alone; and the same 100 labels put **blocking precision at 0.210**. The verifier is good at *is this
> real*. Nothing in the pipeline checks *is this worth blocking* — and no model upgrade would change
> that.

## What this does NOT establish

- **Recall, relative or absolute.** CodeRabbit produced **0 findings on all 19 agent PRs**, so the
  union of confirmed defects is the panel's own set and any relative-recall band collapses to [1, 1] —
  a property of the labelling, not of the reviewer. Absolute recall needs defects found outside both
  arms, which a corpus assembled out of what reviewers found cannot contain. **Permanently
  unmeasurable from this corpus**, and adjudicating more findings does not change it.
- **Minor and nit accuracy.** The channel this data came from carries only `critical|major` with
  `lane != backlog`. Minor and nit findings exist upstream and never reach here.
- **Reliability.** K=1, so no inter-annotator agreement figure. The 84/16 model split is too
  confounded to serve as a second annotator.
- **A simple random sample.** Every finding the fix agent did not record as `fixed` entered with
  probability 1 (a certainty stratum); the rest were stratified by state × severity × lens, seed
  20260820. Quote it as *stratified with a certainty stratum*.

## Invariant 3 — instrument or arm?

> *This is a claim about an ARM. It is not in `reports/` because the lane refuses a K=1 corpus —
> `score-all.mjs:668`, a precondition that belongs to `reliability.mjs` — and `2026-08-20-agent-prs`
> is K=1. **`validity.mjs` is fully wired and has already run**: it is in `score-all.mjs`'s `STEPS`
> at :107 and `report.mjs`'s `SECTIONS` at :139 (#908), and
> `scores/by-config/…__2026-08-10-pilot-reviewed/validity-v1.json` exists — reading `labels.total: 0`,
> because the only corpus the lane will score has 543 **pair** labels and no **finding** labels.
> Roadmap item D is "run it on the corpus that has labels", not "wire it up."*

All four references re-checked 2026-08-22 against `wafflebase/wafflebase` `main` at
`148bfcb4e171de4adbb575ec585f415b09515aee`, and against this repository's own
`validity-v1.json`.

## `raw/`

| file | what it is |
|---|---|
| `raw/dataset.json` | 680 KB, one row per labelled finding: what the reviewer said, what the fix agent did with it, all three adjudication passes with their own `cost_usd`, and the label of record with its `presented_fields` / `withheld_fields` blinding disclosure |

**`dataset.json`'s own `meta` block is authoritative over this README** — it carries `caveats`,
`field_glossary` and `how_the_labels_were_made`, and it travels with the data.

**Not copied, deliberately:**

- **The 100 label files.** They are already in this repository at `labels/2026-08-20-agent-prs/`,
  committed at `fbc8134`. Copying them would create a second copy of a record this store already
  holds.
- **The delivered `report.md`** (625 lines). It is a hand-authored presentation document, it stays in
  the working kit, and the two sections of it that are *defects rather than measurements* are being
  filed upstream separately.
- **The pass-1/2/2b journals.** Every figure above is reproducible from `dataset.json` alone. The
  journals add only the 20 uncharged failure rows (237 journal rows → 217 charged calls) and
  reconcile to the same dollar total.

## ⚠ Corrected against the evidence

The delivered report's verification section states **"Cost: $116.30 across both passes, 200 model
calls."** Summed from `dataset.json` — and independently from the 16 pass-1 journals, the 16 pass-2
journals and the 3 pass-2b journals in the working kit, which agree to the cent — the real figures
are:

```
pass1    100 calls   $48.6268     (diff only)
pass2    100 calls   $71.5951     (diff + cited files)
pass2b    17 calls    $9.3941     (re-judged with better evidence)
         217 calls  $129.6160
```

$116.30 does not match the total, and it does not match pass1 + pass2 ($120.2219) either. **No
reported result changes** — precision, blocking precision and every per-lens figure are computed from
the labels, not the spend. The figure quoted in the room was low by $13.32.
