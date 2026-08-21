# CLI: --quiet discards the result body and the error envelope, leaving failures with no diagnostic

## Summary

`--quiet` suppresses everything, not just progress notices. A successful command produces zero bytes instead of its result, and a *failing* command exits non-zero with **zero bytes on both stdout and stderr** — no diagnostic at all.

## Documented contract

`docs/design/cli.md:926`:

> Text results (json/md/text): stdout by default; `--out` redirects to a file. `--quiet` suppresses progress notices but **preserves the body**.

## Reproduction

No backend or auth needed:

```
$ wafflebase schema           # exit 0, 4797 bytes of JSON
$ wafflebase schema --quiet   # exit 0, 0 bytes          <- body discarded
```

And the worse case — a failure under `--quiet`:

```
$ export WAFFLEBASE_SERVER=http://127.0.0.1:1 WAFFLEBASE_API_KEY=wfb_dummy WAFFLEBASE_WORKSPACE=ws
$ wafflebase docs content d1 --quiet
$ echo "exit=$?"
exit=1
# stdout: 0 bytes
# stderr: 0 bytes
```

Without `--quiet` the same command prints `{"error":{"code":"ERROR","message":"fetch failed"}}`.

## Why it matters

Two separate problems, the second more serious than the first.

**The body is data.** `--quiet` is the natural flag for scripting, and it currently makes every read command return nothing. A caller doing `wafflebase schema --quiet > out.json` silently writes an empty file.

**A non-zero exit with no diagnostic is unactionable.** The caller knows only that something failed — not what, not whether it is retryable. Suppressing *progress* output is a display concern; suppressing the error envelope removes the one machine-readable signal the CLI documents. Errors go to stderr precisely so they survive output redirection and quiet modes.

## Suggested direction

`--quiet` should gate progress notices only, and never the result body or `outputError`. The relevant sink is `packages/cli/src/output/formatter.ts` — `outputError` (`:37-44`) takes `quiet` and honours it; the documented behaviour is that it should not.

---
Found by the CLI issue hunter and hand-verified against `b70302dd1`. Not auto-filed.