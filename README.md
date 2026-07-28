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
reports/<comparison_id>.md   static A-vs-B comparisons
labels/<corpus_version>/<item_id>.json   Track B (validity) — reserved
```

## Invariants

- **`runs/` is write-once.** A re-run is a new `run_id`, never an edit in place.
- **Metrics live in `scores/`, never in `runs/`** — change a matcher/metric and
  re-score cached run artifacts (cheap); no need to re-invoke the model.
- **Identity keys:** `corpus_version` = the input set; `config_hash` = the judge
  composition. `(config_hash, corpus_version)` is the comparability key — same
  pair across `run_id`s = replicates (the substrate for cross-run reliability).
- **Transcripts are gzip-compressed** and trimmed (git-bloat guard).

## Field-level contract

The authoritative field-by-field schema for every file above (config manifest,
`config_hash` canonicalization, run/envelope/payload/score shapes) is maintained
alongside the harness. This README is the layout + invariants; the harness's
schema doc is the source of truth for field definitions.
