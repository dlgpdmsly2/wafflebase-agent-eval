# How much does the adjudicator itself move the numbers?

**Question.** Every precision figure in this project rests on a model's judgement of whether a
finding is a real defect and how bad it is. That judgement was made by **two different models**:
84 of the 100 labels by `claude-fable-5`, 16 by `claude-opus-5` after the first account's quota was
exhausted mid-run. Fable is now unavailable, so **every label from here on is Opus.**

That leaves a confound under everything downstream. A gate change measured with new Opus labels
against the old Fable baseline measures *two* changes at once — the gate, and the swap of grader —
with no way to separate them. Worse, the little evidence available pointed the flattering way:
unpaired and on different pull requests, Fable's blocking precision read 0.226 and Opus's 0.125.

So the two graders were run against each other on identical findings, under a hash-pinned prompt
and identical evidence. **And twice**, because one Opus run is not interpretable on its own: two
adjudications differ partly because the graders differ and partly because *any* two differ. The
second run is what makes the first readable.

**Answer: it moves the severity numbers a great deal and the realness numbers not at all.**

| n=40, same findings | `is_real` | exact severity | **blocking boundary** |
|---|---|---|---|
| Opus A vs Fable | **40/40** | 28/40 · 70.0% | **29/40 · 72.5%** |
| **Opus A vs Opus B — the noise floor** | **40/40** | 35/40 · 87.5% | **37/40 · 92.5%** |
| Opus B vs Fable — consistency check | **40/40** | 27/40 · 67.5% | 28/40 · 70.0% |

On the 21 findings Fable graded blocking, Opus run A demotes **11**, run B demotes **12**, and the
two runs disagree with each other on **3** — a cross-instrument effect **~3.7×** the within-instrument
one. **10 of the 21 are demoted by both runs independently.** Every demotion was `major → minor`:
not one `critical`, not one `nit`, and **zero promotions in either run** out of 19 non-blocking
findings. A systematic offset, not a symmetric disagreement.

| | |
|---|---|
| **verdicts** | **121** — run A 40 · run B 40 · uniform pass 41, plus 6 failed calls at $0 |
| **cost** | **$33.77** — A $13.88 · B $5.27 · U $14.62 ($0.279/verdict) |
| **prompt** | frozen to `raw/prompt-surface.md`, combined `sha256:3ed8234b…67e0ddfc8`, verified by the runner before all 121 calls |
| **model** | `claude-opus-5`, effort `high`, one finding per session |
| **date** | 2026-08-24 |

## 🔵 `is_real` agreed 40/40, three ways — and that splits the two headlines apart

Not one finding changed its real/not-real verdict, between graders or between runs. The finding
survived quadrupling the n: on the 81 findings where both instruments have a verdict, precision
reads **0.951** (Fable) against **0.963** (Opus), each well inside the other's interval.

So the project's two headline numbers behave completely differently under a change of grader:

- **Precision is instrument-independent.** It rests on `is_real`, which has no variance here.
  **It can be carried across the Fable→Opus change and quoted as it stands.**
- **Blocking precision is not.** It rests on severity, which moves systematically. **It cannot be
  compared across the change at all.**

Every disagreement in this probe is about *impact*, never about *truth*. Qualitatively Opus
confirms the mechanism in detail — usually at `high` confidence — and then files it lower:
*"confirms the reading… no scheme allowlist, no loopback/link-local/private-range rejection"* →
`minor`. It is not disputing the defect. It disagrees about what a merge should be blocked for.
**That is a severity-policy disagreement between two competent readers**, which is the same thing
decision 54 concluded about the panel itself.

## What the numbers read once there is only one instrument

The paired result's own recommendation was to put all 100 findings on one grader, so that no
downstream figure is part instrument. That was done — 16 already Opus, 40 from run A (canonical),
41 from a uniform pass over the remainder.

```
MIXED   (84 Fable + 16 Opus)   precision 0.960 [0.902, 0.984]   blocking precision 0.210 [0.142, 0.300]   n=100
UNIFORM (all Opus)             precision 0.969 [0.913, 0.989]   blocking precision 0.124 [0.072, 0.204]   n=97

paired subset, n=81 — both instruments on the SAME findings
  Fable   precision 0.951 [0.880, 0.981]   blocking precision 0.235 [0.156, 0.338]   19 blocking-and-real
  Opus    precision 0.963 [0.897, 0.987]   blocking precision 0.123 [0.068, 0.213]   10 blocking-and-real
  direction: 11 of 21 blocking demoted (52.4%) · 2 of 60 promoted (3.3%)
```

⚠ **Which figure lives where.** This probe holds the uniform estimate, **0.124**. The label store
still holds Fable's labels, so **any scorer reading `labels/` keeps returning the mixed-instrument
0.210** — including `validity.mjs`. That is an accepted consequence of writing no label (below),
not an oversight. Stamping the instrument onto stored figures is a separate open item and is not
done here.

The severity vocabulary shows the mechanism without needing the reasoning:

```
mixed    {minor 76, major 23, nit  1}
uniform  {minor 75, major 14, nit  8}
```

Opus reaches for `nit` **eight times as often** and `major` far less. It is not finding fewer
defects — it grades the same ones lower.

## The uniform pass was justified by its result, not only by its prior

The case for buying the remaining 44 was that the *promotion* rate — the only route by which a
`minor` finding can enter the blocking numerator — was bounded only weakly. The paired sample saw
0 of 19, whose Wilson 95% upper bound is 0.168, so up to ~7 of the 44 could promote.

**Measured over 60 non-blocking findings: 2 promotions, 3.3%.** Not zero. Those two lifted the Opus
blocking-and-real count from 8 to 10 — **a 25% relative move on the headline.** Had the pass been
skipped on the strength of 0/19, the uniform figure would have been understated by a quarter.

📐 **The general form is worth keeping: a 0-of-n at small n is not a measurement, and the interval
is what says so.**

## Where the correction bites — `security`, the lens that was carrying the gate

Per lens, run A against Fable. Rate withheld below n=5.

**Two different denominators, kept apart on purpose** — `sampled` is that lens's share of the 40,
`blocking` is how many of those Fable graded blocking. A demotion can only come out of the second.

| lens | sampled | of those, Fable-blocking | demoted | promoted |
|---|---|---|---|---|
| `security` | 9 | **9** | **4** | 0 |
| `blast-radius` | 9 | 6 | **4** | 0 |
| `correctness` | 11 | 5 | 2 | 0 |
| `design-fit` | 7 | 1 | 1 | 0 |
| `test-adequacy` | 4 | 0 | 0 | 0 |
| **total** | **40** | **21** | **11** | **0** |

No per-lens *rate* is quoted: every blocking cell is under n=10, so the counts are the honest
output. `test-adequacy` contributed 4 sampled findings and no blocking one at all.

`security` held **9 of the 21** blocking findings and **4 of the 10** demotions both runs agreed
on. The validity report named it the only lens above one in three (0.53 blocking precision) and
therefore the one carrying the gate. **Under Opus it carries considerably less.** One slice beyond
the pre-named tables — the lens composition of the both-run demotions — is reported as exploratory.

## 🔴 Three findings have no Opus verdict, and the token was not the problem

`pr-873`, `pr-864` and `pr-899` are unjudged. Six error rows across the run-U journals — three from
the pass, three from a top-up attempt — every one identical:

```
kind=pool-exhausted  retryable=False
"every credential in the pool (1) was retired — nothing left to fail over to"
```

A **fresh credential was minted and genuinely used** (a different credential by hash) and failed the
same way on its *first* call, at **$0**. Both tokens were structurally sound. A freshly minted token
retired on first use means the credential is rejected or its window is closed, and `claude
setup-token` mints for whichever account is logged in: **the usage limit is on the account, not the
token.** A new token for an exhausted account is still exhausted.

⚠ Contributing cause worth recording: `--token-file` **forces a pool of exactly one** — it sets
`CLAUDE_CODE_OAUTH_TOKEN` and deletes `CLAUDE_CODE_OAUTH_TOKEN_1..8`, which is why the error says
*"the pool (1)"* and *"nothing left to fail over to"*. Four concurrent windows on one credential is
what retired it. A real pool needs the env vars and no `--token-file`; that trades the file's
privacy properties for failover and should be an explicit choice, not a default.

### The gap cannot move any conclusion, and that is arithmetic

All three are Fable-`minor`, so they can only enter the blocking numerator by being **promoted**.
At the measured 3.3% the expectation over three findings is ~0.1. Both extremes:

```
none of the 3 promote      12/100 = 0.120
ALL of the 3 promote       15/100 = 0.150   (each would also have to be is_real)
```

**So the uniform blocking precision lies in [0.120, 0.150] whatever they turn out to be — against
the mixed-instrument 0.210 over those same 100 findings, and Fable's 0.235 on the 81 where both
instruments have a verdict.** The gap holds at either extreme. Finish them if a future scorer wants
a clean n=100; **no conclusion here waits on them.**

## Invariant 3 — instrument or arm?

**This is a claim about the INSTRUMENT**, and it is the cleanest example of the category in this
store yet: the subject is the *adjudicator*, not either reviewer. Nothing here says the panel got
better or worse, and no arm is compared to another. It measures whether the ruler changed length
between the two halves of the label set — so `reports/`, which is lane-rendered and keyed by
`comparison_id`, is not where it belongs.

The arm figures it quotes (`0.960`, `0.210`) are the lane's own published numbers, restated as the
baseline being re-measured rather than offered as new arm results.

🔴 **No label was written, and that is a decision rather than an omission.** The store's label path
is keyed off the *finding*, so an Opus verdict for a finding Fable already labelled lands **on
Fable's own file** — after which the old instrument survives only in these journals. That was
declined outright. Two alternatives were considered and are not done: a **second corpus version**
(`labels/<cv>-opus/`) so both instruments coexist and `validity.mjs` could score either — its own
scoped task, and it has not been verified that a second version can be created without
re-extracting; and **relabelling in place**, which is irreversible and was rejected.

## What this does NOT establish

- 🔴 **Neither adjudicator is ground truth.** The output is the size of a gap between two
  instruments, not a ruling on which one is right. Nothing here says the panel's findings deserve
  blocking or do not — it says the answer depends on who is asked, by a margin that swamps the
  noise.
- 🔴 **Family sympathy is unmeasurable here, not absent.** The panel's lenses run on
  `claude-opus-5`/`claude-sonnet-5`, so Opus grading Opus-authored findings shares a model family
  with its subject. A grader systematically sympathetic to reasoning shaped like its own cannot be
  detected from inside this probe — **and the one axis where it would show carries no information,
  because `is_real` has zero variance.** A field that never disagrees cannot separate "both graders
  are right" from "both graders are wrong the same way". Opus being the *stricter* of the two is
  suggestive against a naive sympathy effect and no more; strictness on severity and sympathy on
  truth are different axes, and only the first was measurable. **Fable being a different model
  family was a methodological property, and it is now permanently lost.**
- **The prompt's provenance is inferred on one leg.** The driver was edited between the Fable
  labels and this probe; the edit is identified as non-model-visible, but no baseline hash existed
  before `raw/prompt-surface.md`. The within-Opus noise floor is immune — both sides used the same
  surface. The cross-instrument leg is not, and cannot be formally cleared.
- **n=21 on the arm that matters.** The load-bearing claim is not the 52.4% rate but the
  **3.7× ratio against the noise floor** and the **10 of 21 demoted by both runs independently**,
  which is a stronger statement than one rate.
- **The 0.37–0.42× retention factor is not a correction to apply.** It is not uniform across lenses
  (`security` 5/9 against `correctness` 9/11) and a single scalar would carry false precision. The
  fix is a uniform instrument, which is what the assembled dataset is for.
- **Minor/nit accuracy, absolute recall, and the miss profile** remain unmeasurable from this
  corpus, unchanged by anything here.

## `raw/`

| file | what it is |
|---|---|
| `raw/prompt-surface.md` | the entire model-visible surface — system prompt, verdict schema, card construction, evidence condition — frozen with per-region and combined sha256. The runner verified these before every call and refuses on mismatch |
| `raw/journal-runA-shard{0,1}.jsonl` | 40 rows → **40 verdicts**, $13.88, 0 failures. Run A is **canonical** for the assembled dataset |
| `raw/journal-runB-shard{0,1}.jsonl` | 40 rows → **40 verdicts**, $5.27, 0 failures. The noise floor — **deliberately not merged** into the dataset |
| `raw/journal-runU-shard{0,1,2,3}.jsonl` | 47 rows → **41 verdicts** + **6 failures at $0**, $14.62. Shards 2 and 3 carry the top-up attempt's rows as well as the original pass's |
| `raw/dataset-opus.json` | the assembled uniform instrument: 97 of 100 findings with an Opus verdict, its provenance per finding, the three named gaps, and the figures above |

Every journal row records what that session was shown — `saw_diff`, `saw_cited_files`,
`cited_unresolved`, `cached_prefix` — beside its verdict, and carries the Fable label it is being
compared against, so the comparison needs no second join. Reconciled across all eight: **121
verdicts, zero duplicate keys within any run, every assigned finding either answered or recorded as
a failure.** Run B is byte-separable from run A by construction: separate processes, separate
journals, one finding per session.

**Run B's cost is not an error.** $5.27 against run A's $13.88 for the identical workload and the
identical cold/warm split (23/17) — the two passes ran concurrently over identical prefixes, so run
B read the prompt cache run A had just written. Cache sharing is not an independence leak: it reuses
input-token computation and carries no model output.

**Not copied, and why:**

- **The per-run and top-up `.json` summaries.** They restate the enumeration already recoverable
  from the journals and derive their roll-ups from them, so they are re-derivable at $0 — and
  unlike the journals they record a local absolute credential path in `token_file`. The top-up's
  entire outcome is the 3 `pool-exhausted` rows already in the run-U shards.
- **The `.log` files and the `.mjs` harness, its tests and its mutation harness.** Tooling, not
  evidence; they stay in the kit. The harness cost $0 to write and can be re-run against `raw/`.
- **The source dataset.** Already filed byte-identically at
  [`../2026-08-20-agent-pr-validity/raw/dataset.json`](../2026-08-20-agent-pr-validity/raw/dataset.json),
  `sha256:c5711d2d5efb8e87f53d95d1…` — linked rather than duplicated.
- **The Fable-era journals.** They belong to the run that produced the 100 labels, not to this
  probe; the Fable verdict each comparison uses travels inside every journal row here as
  `fable_label`.
