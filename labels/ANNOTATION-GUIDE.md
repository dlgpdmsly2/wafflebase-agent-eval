# Annotation guide — Track B validity labels

*Governance doc for the labeled dataset under `labels/<corpus_version>/`. This is
the B0 "annotation guide" deliverable: the definitions that let two annotators
fill the same fields the same way, and therefore the precondition that makes
inter-annotator agreement (B3) meaningful. Companion to the label schema frozen
in `store.mjs` (`putLabels` / `putFindingLabel`) and the design rationale in
`labeled-dataset-validity-overview.md` §5.*

---

## 0. What a label is, and is not

A label is **ground truth about the code** — "is this a real defect, and what is
the correct verdict on this PR" — recorded **independently of what the review
panel said**. The panel's output is the thing we are grading; using it to write
the label is circular (overview §4). A label answers *reality*, not *did the
panel agree with me*.

Two hard rules follow:

1. **Adjudicate blind to the panel.** Read the diff at its review point and decide
   for yourself before looking at the panel's findings/verdict. If you must see
   panel output (e.g. to know which findings reached the verifier), record your
   own `is_real` judgement from the *code*, not from whether the panel kept it.
2. **A merge is not a label.** "A human merged it / never complained" is one noisy
   vote, not truth — authors accept or ignore review comments for reasons
   unrelated to correctness (deadline, ownership, politeness; overview §3.1).
   Never set a label from merge status alone.

---

## 1. The two records

Labels live under `labels/<corpus_version>/`, keyed exactly like the corpus item.

### 1.1 Item-level — `labels/<cv>/<item_id>.json`

The PR-level verdict truth (feeds gate validity, V3).

| Field | Values | Meaning |
|---|---|---|
| `item_id` | `pr-N` | matches the corpus item |
| `corpus_version` | e.g. `2026-07-28-pilot` | the corpus this label is scoped to |
| `verdict_label` | `block` \| `approve` \| `borderline` | the correct gate verdict (§3) |
| `primary_defect_class` | a lens id, or `null` | dominant defect kind; `null` when `approve` |
| `true_defects[]` | see §4 | every real defect the PR contains (incl. ones the panel missed) |
| `stratum` | `benign` \| `known-defect` \| `reverted` | corpus stratum, for slicing (§5) |
| `label_source` | `gold` \| `silver` \| `distant` | how the label was produced (§6) |
| `annotators[]` | ids | who adjudicated, for IAA bookkeeping (§7) |
| `confidence` | `high` \| `medium` \| `low` | annotator's certainty (§7) |
| `evidence` | free text | *why* — the concrete basis (check-runs, revert PR, code read) |
| `diff_sha256` | `sha256:…` | the item's `sha256_diff` at label time; drift guard (§8) |
| `notes` | free text | caveats, scope, anything a second annotator needs |

### 1.2 Finding-level — `labels/<cv>/findings/<item_id>/<sha256(finding_key)>.json`

Truth about **one specific finding the panel raised** (feeds detection precision
V2 and verifier validity V1). Keyed by `finding_key` = the panel's own
`findingKey(f)` = `` `${file}::${lowercased summary}` `` — the same identity the
verifier artifact carries, so the label joins on with no fuzzy matching.

| Field | Values | Meaning |
|---|---|---|
| `finding_key` | `file::summary` | the panel's key (stamped in by `putFindingLabel`) |
| `is_real` | `true` \| `false` | **is this an actual defect** — the core judgement (§2) |
| `should_verifier_keep` | `true` \| `false` | should the verifier keep it (§2); normally `= is_real` |
| `severity` | `critical`\|`major`\|`minor`\|`nit` | your severity, not the panel's (§4.1) |
| `kind` | lens id / defect class | `correctness`, `test-adequacy`, … |
| `label_source`, `annotators`, `evidence`, `notes` | as above | |

---

## 2. `is_real` and `should_verifier_keep` — the two finding judgements

**`is_real`** — *does the described defect actually exist in this code?*

- `true`: the finding points at a genuine problem in the diff (even if minor).
- `false`: a **hallucination** — the described problem is not present. The claim
  is factually wrong (the guard it says is missing is present; the crash it
  predicts cannot occur; the file it cites does not exist).

`is_real` judges **existence, not importance.** A real-but-trivial nit is
`is_real: true, severity: nit` — *not* `false`. "False" is reserved for claims
that are *wrong about the code*, because that is what a hallucination is. This
distinction is what keeps the verifier's confusion matrix honest: dropping a
correct-but-trivial finding is a different event from dropping a fabrication.

**`should_verifier_keep`** — *should the verifier have kept it?* Normally equal
to `is_real` (keep reals, drop fakes). It diverges only in deliberate edge cases
— e.g. a finding that is technically real but so far out of scope it should be
dropped. When it diverges from `is_real`, **`notes` must explain why**; otherwise
omit the divergence and let it mirror `is_real`.

---

## 3. `verdict_label` — the correct gate verdict

The shipped gate is a pure function (`severity.mjs`): **APPROVE iff the PR has
zero `critical`/`major` real defects**; otherwise BLOCK. Label the *correct*
verdict under that same rule, from your `true_defects`:

- `block` — the PR contains ≥1 real `critical`/`major` defect that a reviewer
  should stop the merge on.
- `approve` — no real blocking defect. Minor/nit observations may still exist;
  they do not block. This is the **benign / true-negative** case.
- `borderline` — genuinely defensible either way (a major-vs-minor severity call
  a reasonable reviewer could go both ways on). Use sparingly; `notes` must say
  what the tension is. Borderline items are excluded from strict scoring and
  reported separately.

`verdict_label` must be **consistent with `true_defects`**: `block` ⟺ at least one
`true_defect` is `critical`/`major`. An `approve` with a major `true_defect` is a
contradiction — fix one or the other.

---

## 4. `true_defects[]` — the defect set

Every real defect the PR contains, **including defects the panel never found**
(those have no `finding_key`; they are what detection *recall* is measured
against). Each entry:

```jsonc
{ "file": "packages/…/x.ts",
  "line_range": [start, end],      // localization unit, §4.2
  "severity": "major",             // §4.1
  "kind": "correctness-regression",
  "description": "what is wrong and why it is a defect" }
```

For a `benign` PR this is `[]` — the affirmative claim "I read the diff and found
no real defect," not merely "I didn't look."

### 4.1 Severity (reuse `severity.mjs`)

`critical | major | minor | nit`. **critical/major are blocking; minor/nit are
not.** Rough calibration:

- `critical` — data loss / corruption, security hole, crash on a common path.
- `major` — incorrect behavior on a realistic path; a regression of existing
  behavior; the feature's core promise is broken. **Blocks.**
- `minor` — real but narrow: an edge case, a missing-but-non-critical test, a
  degraded (not broken) behavior. Does not block.
- `nit` — style, naming, a check-before-merge note with no demonstrated impact.

When unsure between `major` and `minor`, the deciding question is *"would a
reviewer be right to stop the merge on this alone?"* If yes → major. If it is a
genuine coin-flip, consider `verdict_label: borderline` and say so in `notes`.
(For weighted metrics, overview §5.6 uses critical=4/major=2/minor=1/nit=0.5.)

### 4.2 Localization unit

`line_range` is `[start, end]` **in the diff's new-file line numbers** for the
primary site. Point at the *cause*, not every symptom site. If the defect is not
line-localizable (a missing test, an absent guard), use the range of the code
that *should* have changed and explain in `description`.

---

## 5. Stratum

Record the corpus stratum so metrics slice by it (overview §5.3). A random PR
sample is mostly "fine" and under-exercises the panel; the strata are what put
real defects and clean code deliberately on both sides.

- `benign` — no real defect; the correct verdict is `approve`. Measures the
  **cry-wolf / false-alarm rate on clean code** — a failure mode in its own right.
- `known-defect` — a documented real problem is present (`block`).
- `reverted` — later reverted / hot-fixed, i.e. it *did* contain a defect review
  should have caught (a strong recall positive; `block`).

---

## 6. `label_source` — the trust tier

- `gold` — a qualified **human** read the diff blind and adjudicated (optionally
  corroborated by hard external evidence, e.g. failing CI check-runs). Highest
  trust. The IAA ceiling is computed over gold labels.
- `silver` — a weaker adjudication: an AI read-through, or several noisy signals
  merged with disagreements resolved, **pending human confirmation**. Usable but
  imperfect; do not treat as the ceiling.
- `distant` — a label inferred from a natural experiment without per-item reading
  (a revert PR, an injected bug). Cheap, coarse.

Be honest about the tier. An AI-adjudicated read-through is `silver`, not `gold`,
even when confident — reserving `gold` for human adjudication is what keeps the
IAA ceiling (B3) trustworthy. See §7 on attribution.

## 7. `annotators` and `confidence`

- `annotators[]` records **who actually adjudicated** — a human id for gold, the
  model id (e.g. `claude-opus-4-8`) for an AI read-through. Attribute to the real
  adjudicator; do not launder an AI read into a human id, or a future IAA pass
  will treat two AI reads as independent human agreement.
- `confidence` is the annotator's own certainty *given their evidence*:
  `high` (clear-cut, or corroborated), `medium` (read-through of a larger diff
  with no ability to run it), `low` (thin evidence; flag for a second look).
  Note that **absence-of-defect is inherently harder to prove than presence** — a
  `benign` label over a large diff should rarely be `high` without tests or a
  captured clean verdict backing it.

## 8. Drift guard

Stamp the item's current `sha256_diff` (from its corpus `meta.json`) into
`diff_sha256`. `store.labelStatus()` returns `stale` when the corpus is
re-extracted and the diff changes, so the scorer drops a label written against a
diff that no longer exists rather than scoring the wrong code. Re-adjudicate and
overwrite (labels are correctable; the store overwrites on `put`).

---

## 9. Worked examples in this dataset

- `pr-521` (**gold**, `block`, `known-defect`) — human-adjudicated against the
  real PR correctness check-runs; the `prior-round` verifier false-negative seed.
  See its item + finding labels.
- `pr-493`, `pr-502`, `pr-471` (**silver**, `approve`, `benign`) — AI
  read-through true-negatives (§ this task). Each is a clean, tested, merged
  change; `true_defects: []`. `pr-493` additionally has a captured panel approval
  corroborating it (used as corroboration, not as the label — §0).

---

## 10. Pair labels — "are these two findings the same defect?"

*Added 2026-08-13. A **pair label** is a different record from §1's two: it judges a
PAIR of findings from different arms, and it lives at
`labels/<cv>/pairs/<pair_key>.json`. It exists because `matchFindings` never merges a
`maybe`, so every undecided cross-arm pair counts as two unique catches and the
complementarity overlap is reported as a band. A label resolves the pair; it does not
resolve a threshold (decision 24 stands).*

### 10.1 The key

`pair_key` is `sha256(file|line|summary of BOTH sides)`, truncated to 12 hex —
`inspect-maybes.mjs`'s `pairKey`. **Deliberately not `finding-match.mjs`'s
`contentDigest`**, which keys a single finding and is the matcher's business: a human
label's key must not move when the matcher is refactored.

⚠ **The key DOES move when a finding's text is re-parsed.** `wafflebase#801` repaired a
CodeRabbit title (`/node_modules/` → a real sentence) and six of the first 23 pair keys
changed. Records therefore carry **both** `pair_key` and `pair_key_at_801`, and a
consumer must index on both — otherwise it will write a second label for a pair that
already has one, under the other vintage's key, and count the pair twice.

### 10.2 The question, and the test that decides it

> **`same`** — the two findings are about the SAME DEFECT. Not *"both are about this
> file"*, not *"both are worth fixing"* — the same underlying problem, however
> differently worded.
> **`different`** — different defects that happen to share a location.
> **`insufficient-basis`** — you would need to read the code to tell. Use it freely.

**The operational test, agreed 2026-08-12:**

> **After you make the edit one reviewer is asking for, does the other reviewer's comment
> still stand?** Discharged too → **one defect**. Survives → **different defects that
> share a location**.

Three riders, each from a real case:

1. If a reviewer offers **two alternative fixes**, either one discharging the other
   comment makes it `same`.
2. **Contradictory prescriptions to the same lines are `different`** — no single edit
   satisfies both.
3. If whether the fix discharges the other comment **depends on a fact in the code that
   neither text states**, the answer is `insufficient-basis`, not a guess.

### 10.3 Worked examples

*These are the cases where two annotators actually diverged, decided explicitly. They are
the rubric — the sentence above is only its summary.*

**A. `same` — different lines, one edit.** *(pr-549, `list-empty-bullet-plugin.ts`)*
Panel at `:182`: *"silent shortcut omits upstream `list`'s 4-space indented-code guard"*,
its evidence naming `isEmptyBulletLine` and the missing `sCount - blkIndent >= 4` check.
CodeRabbit at `:31`: *"missing over-indent guard in `isEmptyBulletLine`"*.
→ **`same`.** Add the `>= 4` check to `isEmptyBulletLine` and both comments are answered.
**Different line numbers are not evidence of different defects** — one names the caller,
the other the helper.

**B. `different` — one code branch, two defects.** *(pr-549, same file)*
Panel at `:196`: the silent branch's paragraph-interruption behaviour *is only tested
inside a list*. CodeRabbit at `:31`: the *missing over-indent guard* (the same finding as
A).
→ **`different`.** Adding the guard does not add the test; adding the test does not add
the guard. **This is the subtle form of the "both are about this file" exclusion: same
mechanism, same branch, two edits.** Note that A and B pair the SAME CodeRabbit finding
with two different panel findings and get different answers — that is the labeler
working, not drifting.

**C. `different` — contradictory prescriptions.** *(pr-429, homepage todo)*
CodeRabbit at `:39`: tick the checkboxes for lines 62–70, *the work is done*. Panel at
`:69`: work item 3 in that range *was never implemented*.
→ **`different`.** Both say the doc misstates reality, and they say it in opposite
directions. No single edit satisfies both, so they cannot be one defect — even though
they concern overlapping lines of one file.

**D. `different` — same file, disjoint claims.** *(pr-429, same doc)*
CodeRabbit at `:1`: the status field still reads `planning`, and four export paths are
missing package prefixes. Panel at `:69`: the unimplemented work item.
→ **`different`.** Genuinely arguable — "the task doc is stale" is one theme — but the
edits are disjoint and neither discharges the other.

**E. 🔴 The recorded disagreement — read this one before trusting the rule.**
*(pr-605, `yorkie-note-store.ts`)*
CodeRabbit at `:48`: keep the test-only undo-stack accessor out of `canUndo()` so
production uses the public `doc.history.canUndo()` contract. Panel at `:106`: `undoFloor`
is an absolute stack depth against a stack that can shrink, so undo stops early. (And at
`:49`: no `markUndoBaseline()` re-base hook.)

- **Read as `same`:** the panel's own evidence says the floor's consumer *is* `canUndo()`.
  Take CodeRabbit's edit and `canUndo()` stops consulting the depth, so the symptom goes.
- **Read as `different`:** these are two problems in one *mechanism* — an encapsulation
  violation and an unsound comparison — and "same mechanism" is close to the *"both are
  about this file"* case the rule excludes. CodeRabbit's remedy is about which contract is
  read, not about the floor's soundness.

**The stored label is `same`** (`058344a36138`, `62003b7ce8c2`), held by the human on
re-read. **Under the rule as written the narrower `different` reading is at least as
faithful**, and the fact that would settle it — *does anything other than `canUndo()`
consume `undoFloor`?* — **is in neither text**, which by rider 3 makes a strong case for
`insufficient-basis`. Recorded rather than resolved. **This pair is the best candidate in
the set for a second annotator.**

### 10.4 What must not change

**Rejecting unrelated pairs confidently.** On the first blind run the `different` class
was the one both annotators agreed on completely — **5 of 5**, including three pairs where
one CodeRabbit finding fans out across three panel findings in a *different file*, admitted
to the queue only by `locationScore`'s 0.6 basename-agreement fallback. Any future
sharpening of this rubric must leave that behaviour intact. **A labeler that never says
`different` is broken**, and roughly half of a real queue is same-file-different-subject.

### 10.5 Provenance for pair labels

As §6–§8, with two additions:

- **`rubric`** — which definition was applied. `one-fix-discharges-both` from 2026-08-12;
  labels made before that carry the original worksheet wording and are marked
  `worksheet-original` under `supersedes`.
- **`supersedes`** — when a verdict is re-adjudicated, the previous verdict, its rubric
  and its basis are kept, not overwritten. §8 says re-adjudicate and overwrite; that is
  right for the *verdict* and wrong for the *audit trail*, because two published
  agreement numbers against two gold vintages are only checkable if both are visible.
