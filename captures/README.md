# Live stage-detail captures

**Machine-written. Do not hand-edit.** Everything here is copied verbatim out of
GitHub Actions artifacts by `scripts/agent/collect-captures.mjs` in
`wafflebase/wafflebase`, via `.github/workflows/capture-collect.yml`.

## Why this exists

The review panel records what each lens did on every review and uploads it as an
Actions artifact. **GitHub deletes those after 90 days**, and `expires_at` is
fixed at upload — no setting rescues one that already exists. This directory is
the copy of record.

## The layout

    stage-detail/channel=<gating|advisory>/pr=<n>/sha=<8hex>/run=<id>/attempt=<n>/
        meta.json          attribution, written by the producer (wafflebase#673)
        <lens>.json        that lens's stage-detail, verbatim

`run=` and `attempt=` are in the path so a path names exactly **one execution**.
Two consequences worth knowing: writes are idempotent, so re-running the
collector is harmless; and two review rounds of the same PR stay two
measurements instead of collapsing into one.

`channel=` separates a gating round from an advisory one. The same commit can
legitimately have both, and they are different measurements.

`key=value` segments are the Athena partition convention — this is the same key
scheme the eventual S3 store uses, so migrating is `aws s3 sync`.

## Two rules

**Write-once.** A path's contents never legitimately change. The collector
refuses to overwrite; to correct something, delete and re-collect.

**`meta.json` present ⟺ the capture is complete.** It is written last, so a
directory holding lens files but no `meta.json` was interrupted and should be
re-collected rather than trusted.

## What is not here

Captures produced before wafflebase#673 carry no `meta.json` and are
**unattributable** — the collector refuses them rather than guessing which PR
they belong to. Any rescued by hand are marked `"provenance": "manual-backfill"`
so they are never mistaken for automatic ones.
