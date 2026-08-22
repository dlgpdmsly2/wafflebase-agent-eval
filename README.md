# wafflebase-agent-eval

Data + provenance store for offline evaluation of the wafflebase autonomous
agent pipeline. **Data only — no code.** The harness lives in the product repo at
`scripts/agent/eval/` and reads/writes this repo through an `ArtifactStore`
interface, so this store is relocatable (a config value, not a hard dependency).

Named generally (`agent-eval`, not `review-eval`) so multiple evaluation
**targets** can share it: the review panel today; a code-fixer or end-to-end
pipeline target later, each as a new adapter writing the same artifact shapes.

## Layout

```
corpus/                 frozen input sets (versioned, immutable)
  manifest.json         corpus_version + item index + checksums
  items/<item_id>/      one diff per item: meta.json, diff.patch, changed-files.txt, issue-spec.md?
configs/<config_id>.json   judge composition manifest (config-as-code)
runs/<run_id>/          IMMUTABLE once written
  run.json              run-level envelope + totals + status
  config.snapshot.json  resolved config + inlined rubric text (reproduction receipt)
  items/<item_id>/      envelope.json, payload.json, transcript.json.gz
scores/                 metrics — RE-SCOREABLE, never written under runs/
  per-run/<run_id>/<scorer_id>.json
  by-config/<config_hash>__<corpus_version>/<scorer_id>.json
reports/<comparison_id>.md   static A-vs-B comparisons, lane-rendered
labels/<corpus_version>/<item_id>.json   Track B (validity) — reserved
probes/<date>-<slug>/    one paid question, its answer, and the evidence
  README.md              the conclusion — renders on GitHub when the folder is opened
  raw/                   the evidence: per-verdict journals + frozen inputs
```

## Invariants

- **`runs/` is write-once.** A re-run is a new `run_id`, never an edit in place.
- **Metrics live in `scores/`, never in `runs/`** — change a matcher/metric and
  re-score cached run artifacts (cheap); no need to re-invoke the model.
- **Identity keys:** `corpus_version` = the input set; `config_hash` = the judge
  composition. `(config_hash, corpus_version)` is the comparability key — same
  pair across `run_id`s = replicates (the substrate for cross-run reliability).
- **Transcripts are gzip-compressed** and trimmed (git-bloat guard).
- **A probe is a question somebody paid to answer, filed so nobody pays again.** Every probe names
  its price and its verdict count. If it cost nothing it is a script, and it stays in the kit.
- **`raw/` is immutable; the conclusion is not.** Nobody outside holds the only copy, so revising a
  reading does not create two documents with one name — and a reading here has already needed
  revision. **A superseded conclusion says so at the top and names what superseded it.**
- **Every probe README must answer: is this a claim about the instrument, or about an arm?** If about
  an arm, **it must say why it is not in `reports/`** — which is lane-rendered and keyed by
  `comparison_id`. This is what stops `probes/` becoming a shadow report directory that routes around
  the scoring lane. It is a sentence each probe owes, not a wall between two folders.

## Field-level contract

The authoritative field-by-field schema for every file above (config manifest,
`config_hash` canonicalization, run/envelope/payload/score shapes) is maintained
alongside the harness. This README is the layout + invariants; the harness's
schema doc is the source of truth for field definitions.
