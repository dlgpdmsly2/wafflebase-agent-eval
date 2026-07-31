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
