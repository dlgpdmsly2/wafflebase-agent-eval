# CLI: the error envelope is multi-line pretty JSON and omits the documented `command` field

## Summary

The error envelope is emitted as pretty-printed JSON spread over several lines and never carries the documented `command` field. The docs specify a single line, and the field is what tells an agent *which* call failed.

Distinct from #654 and #655, which are about the envelope being **bypassed** (bare prose from the arg parser; an upstream body forwarded verbatim). This one is about the shape the CLI emits when the envelope path *is* taken correctly.

## Documented contract

`packages/cli/README.md:109-110`:

> **Errors**: a single JSON line on stderr — `{"error":{"code":"…","message":"…"}}`.

`docs/design/cli.md:930-931` adds the field:

> `{"error":{"code":"…","message":"…","command":"docs.content"}}`

and `docs/design/cli.md:691-698` shows the same three-key shape.

## Reproduction

```
$ export WAFFLEBASE_SERVER=http://127.0.0.1:1 WAFFLEBASE_API_KEY=wfb_dummy WAFFLEBASE_WORKSPACE=ws
$ wafflebase docs content d1 --pages garbage 2>&1
{
  "error": {
    "code": "ERROR",
    "message": "fetch failed"
  }
}
```

Five lines, 71 bytes, no `command`. Expected one line carrying `command: "docs.content"`.

## The code

`packages/cli/src/output/formatter.ts:43-44`:

```ts
console.error(
  JSON.stringify({ error: { code: errorCode(error), message } }, null, 2),
```

`null, 2` produces the multi-line form, and the object literal has no `command` key — there is nowhere for the command name to come from, since `outputError` does not receive it.

## Why it matters

Line-orientation is not cosmetic here. A single JSON line per error is what lets a caller read stderr with a line-delimited parser, interleave it with other output, and match one error to one command. Multi-line output forces a consumer to buffer to EOF and guess at boundaries — and if two errors are ever emitted, they cannot be separated at all.

The missing `command` is the bigger gap for the documented audience: an agent driving several calls sees `{"code":"ERROR","message":"fetch failed"}` with no way to attribute it.

## Suggested direction

Drop the `null, 2` argument, and thread the command name into `outputError` so it can populate `command`. Commander exposes it via the action handler's command object, so the call sites already have it in scope.

---
Found by the CLI issue hunter and hand-verified against `b70302dd1`. Not auto-filed.