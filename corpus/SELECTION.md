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

**Excluded** — the entire `agent/*` local-CLI set (18 PRs) is agent-infra
self-development, not product work; the 3 mega product diffs (#508 is agent-infra
anyway; #503/#500/#499/#480 etc. product megas) as cost outliers.
