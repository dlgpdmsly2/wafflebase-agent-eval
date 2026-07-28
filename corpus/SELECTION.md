# Corpus selection rationale

## 2026-07-28-pilot (20 items)

**Goal:** a reliability corpus — representative of the *kind of diff the review
panel reviews in production*. That criterion is **product code that resolves a
real issue** (Sheets/Docs/Notes/Slides features & bug-fixes), at realistic size.

**What does NOT matter here:** who authored the diff. Reliability measures the
*reviewer's* self-consistency when it re-reviews a FIXED diff K times — the diff's
origin (bot vs human) is irrelevant to that variance. So the corpus is
author-agnostic. (Provenance is still *recorded* per item, because it matters
later for the validity phase — comparing the harness verdict to a diff's actual
production panel verdict needs bot-authored diffs — and it lets us ask the
secondary question "is the panel equally reliable on bot vs human diffs?")

**Selection criteria (기준):**
1. **Product-issue-resolving only** — EXCLUDE agent/pipeline meta-PRs (review
   panel, metrics, prompts, state labels, model config, CI-for-agent). Those are
   "feature-adds to the agent," not what the panel reviews in production.
2. **Representative size spread** — S/M/L mix; exclude mega-diffs (>~1k lines) as
   cost outliers and atypical review targets.
3. **Verdict diversity** — a mix likely to block vs approve (κ is undefined with
   no variance); true verdicts come from the run, this is an a-priori spread.
4. **Some with linked issues** — exercises the design-fit lens (11/20 here).

**Provenance breakdown:** 7 autonomous (`app/yorkie-agent`, issue→PR bot) +
13 human. Reliability is computed over all 20; the autonomous subset can be
reported as a secondary cut.

**Selected (20):**
- Autonomous product (7): 549 548 547 546 544 521 513
- Human product (13): 556 536 533 532 531 523 520 514 507 505 502 493 471

## Scope coverage (recorded per item as `scope`/`additions`/`deletions`)

Range 29–888 changed lines. Buckets (S ≤50 / M ≤300 / L >300, same rule as the
pipeline's effort summary): **L=10, M=9, S=1**.

- **M and L are well-covered and evenly split** — this is the realistic review
  workload; production diffs (incl. the autonomous PRs, 61–888 lines) skew M/L, so
  the skew is representative, not a defect.
- **S is a single point (pr-493) — accepted, and low-cost to under-sample.** Genuine
  product diffs ≤50 lines are rare (sub-50 changes are usually trivial tweaks). More
  importantly, small diffs are where reliability is *least* interesting: they carry
  few/no findings, so both runs almost always **approve**, i.e. the verdict is
  trivially stable. Reliability failures (flips) concentrate in the larger, more
  ambiguous diffs where the panel is on the fence — which M/L covers. So the thin S
  bucket costs little signal.
- Caveat (why we still recorded scope): "small ⇒ approve ⇒ stable" is a reasonable
  assumption, not a law — a small diff can hide a subtle critical bug. The `scope`
  field lets us verify this by slicing reliability by size once runs exist, and add
  S items later if the assumption looks wrong.

**Excluded** — the entire `agent/*` local-CLI set (18 PRs) is agent-infra
self-development, not product work; the 3 mega product diffs (#508 is agent-infra
anyway; #503/#500/#499/#480 etc. product megas) as cost outliers.
