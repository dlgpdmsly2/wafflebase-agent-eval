# Rubric — `same-root-cause`

**Rubric id:** `same-root-cause`
**Frozen:** 2026-08-22, **before** any adjudication run that uses it.
**Applies to:** BOTH arms — cross-replicate (panel run A vs panel run B) and cross-arm
(panel vs CodeRabbit). Applying one rubric to both is a condition of use, not a
convenience; see the pre-registration below.

This file is written so that someone who does not know what numbers it produced can judge
whether it is the right criterion. It contains no results.

---

## The criterion

> **Two findings are the same when they describe one underlying defect, however many
> places it surfaces. They are the same if a human who has understood one has thereby
> understood the other — even when fixing them takes more than one edit.**

They are **different** when understanding one leaves the other still to be understood:
two distinct root causes, even in the same file, function or feature.

### The distinction this turns on

Under `one-fix-discharges-both`, "a missing null check in A" and "a missing null check in
B" are **different**: fixing one leaves the other standing. Under `same-root-cause` they
are the **same** — one defect (the invariant is not enforced), two sites. That single
class of pair is the entire difference between the two rubrics.

### Worked distinctions

| A | B | `same-root-cause` | why |
|---|---|---|---|
| unchecked `await` in `fetchUser` | unchecked `await` in `fetchOrg` | **same** | one root cause, two sites |
| missing test for the `=` setext branch | missing test for the `-` branch | **same** | one gap, two cases |
| `SET TimeZone` changes query semantics | `SET DateStyle` changes output parsing | **different** | two independent settings, two causes, one file |
| relies on a private ruler field | dependency range is unpinned | **different** | two distinct risks co-occurring in one comment |
| the endpoint can throw a 500 | the endpoint has no test | **different** | a defect and a coverage gap are not one defect |

## Why this rubric

Two reasons. **The first is load-bearing because it is external to us.**

1. **It is the unit the comparison arm already uses.** CodeRabbit emits one finding with
   the other affected files listed beneath it: one root cause, N locations, one item.
   Measuring our panel by a unit CodeRabbit does not use makes the two arms
   non-comparable at exactly the level being compared.
2. **It matches the consumer.** The figure says how much work a review lands on a person.
   One root cause understood once is one item of work even when the patch touches four
   files.

## 🔴 Pre-registration

**This rubric moves the two metrics in OPPOSITE directions, and that is why it is
applied to both arms.**

- **Cross-replicate:** reproducibility rises. **This flatters us.**
- **Cross-arm:** more of our findings count as matching CodeRabbit's, so both directional
  overlap rates rise and **the measured distinctiveness of both reviewers falls.** In a
  report whose thesis is where each reviewer wins, that **costs us.**

Choosing a definition after seeing which way it moves your headline is the pattern that
gets a result dismissed however sound the reasoning. **Uniform application to both arms is
the answer to that objection**: a rubric cannot be self-serving if it was applied
everywhere and cost on one metric what it gained on the other.

Conditions of use, not suggestions:

1. **The justification stands on CodeRabbit's report format and on the consumer, not on
   any outcome.** If either argument is wrong, the rubric is wrong whatever it does to a
   number.
2. **Both arms are reported together or not at all.** A write-up that publishes the
   cross-replicate gain and defers the cross-arm cost is the precise thing this design
   exists to prevent.
3. **Both rates always appear together for each arm**, with the rubric named on every
   figure. `same-root-cause` and `one-fix-discharges-both` answer two questions; neither
   replaces or corrects the other.
4. **This file was frozen before the run and is hashed.** Every shard records the hash, so
   eight processes cannot adjudicate against eight slightly different prompts. Any change
   to the criterion after results exist must be a new rubric id with a new file, never an
   edit to this one.

## The structured basis field

Required on every verdict — the mirror of the `basis` field used under the strict rubric.

| value | meaning | used with |
|---|---|---|
| `same-defect-one-site` | one defect at one location, worded differently by the two sources | `same` |
| `same-root-cause-multiple-sites` | one root cause surfacing in more than one place; fixing it takes more than one edit | `same` |
| `different-root-cause` | same file, function or feature, but two genuinely distinct causes | `different` |
| `unrelated` | the two findings concern different problems | `different` |
| `n/a` | the verdict was `insufficient-basis` | `insufficient-basis` |

**Why the split inside `same` matters:** "one defect, two wordings" and "one root cause,
several sites" are materially different claims. The first says the two sources found the
identical thing; the second says they found different facets of one thing. Only a field
distinguishes them afterwards, and the second is the class this rubric exists to
reclassify — so if the merges are overwhelmingly of the second kind, that is a result
about the rubric and must be visible rather than inferred from prose.

## `insufficient-basis`

It went **unused in 381 verdicts** across two prior passes under the strict rubric,
despite being in the schema and named in the prompt as not a synonym for `different`. A
three-verdict schema is not carried into a third rubric unchanged. Both prompts now name
the checkable circumstance in which it is the **required** answer:

> If either finding's text does not identify a *mechanism* — it names only a location, or
> restates the code without saying what is wrong with it — then their root causes cannot
> be compared, and the answer is `insufficient-basis`.

Whether that changes the rate is itself reportable. If the count is still zero, the honest
conclusion is that a three-verdict schema is being run as two, and that must be stated
rather than left for a reader to notice.

## Storage

**No verdict under this rubric may be written to the label store.** `pairLabelKey` hashes
`file|line|summary` on both sides and deliberately nothing else — the key is documented as
rubric-blind so that a human's verdict survives a matcher refactor. Consequently a
`same-root-cause` verdict for a pair produces the **identical filename** as its
`one-fix-discharges-both` verdict, and writing it would overwrite the existing label.
Verdicts live in `reference/` artefacts only. If this rubric survives contact with the
data, the storage fix is a rubric-scoped path (`labels/<corpus>/pairs/<rubric>/`) — **not**
a key change. That is a later PR and a later decision.
