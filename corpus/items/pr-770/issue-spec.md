# CLI: upstream error bodies are forwarded verbatim as if they were the error envelope

## Summary

Six command paths test `if (body?.error)` for *truthiness* before forwarding an upstream error body verbatim. Any JSON error body with a top-level `error` field passes — including every Express/Nest 404 and 500, where `error` is a **string** (`"Not Found"`), not the documented `{code, message}` object. The CLI then prints that body as though it were the documented envelope.

## Documented contract

`packages/cli/README.md:109-110` — a single JSON line on stderr, `{"error":{"code":"…","message":"…"}}`, where `code` is what agents branch on.

## The code

`packages/cli/src/commands/docs.ts:189-198`:

```ts
const res = await getClient(opts).getDocContent(docId);
if (!res.ok) {
  const body = res.data as { error?: { code?: string; message?: string } } | null;
  if (body?.error) {
    // Surface backend-shaped errors (e.g., TYPE_MISMATCH) verbatim
    // so agents reading stderr can act on the `code` field.
    console.error(JSON.stringify(body, null, 2));
    process.exitCode = 1;
    return;
  }
  throw new Error(`HTTP ${res.status}`);
}
```

The cast asserts the shape; the guard never checks it. The intent in the comment — surface *backend-shaped* errors so agents can read `code` — is exactly right, but truthiness does not establish "backend-shaped".

Same pattern at:
- `packages/cli/src/commands/docs.ts:192`, `:254`
- `packages/cli/src/commands/slides.ts:151`, `:204`
- `packages/cli/src/commands/notes.ts:145`, `:202`

## Reproduction

Fully local — a stub returning an Express-shaped 404:

```js
// stub-404.mjs
import http from "node:http";
const BODY = { message: "Cannot GET /api/v1/...", error: "Not Found", statusCode: 404 };
http.createServer((_q, s) => {
  s.writeHead(404, { "content-type": "application/json" });
  s.end(JSON.stringify(BODY));
}).listen(59997, "127.0.0.1");
```

```
$ node stub-404.mjs &
$ WAFFLEBASE_SERVER=http://127.0.0.1:59997 WAFFLEBASE_API_KEY=wfb_dummy \
  WAFFLEBASE_WORKSPACE=ws wafflebase docs content doc-1
{
  "message": "Cannot GET /api/v1/...",
  "error": "Not Found",
  "statusCode": 404
}
```

Expected: `{"error":{"code":"…","message":"…"}}`. Actual: the upstream body, unmodified, with `error` as a string and no `code` field at all.

This is also reachable against a real deployment — any request that misses a route (for example when no workspace is configured, which produces `/api/v1/workspaces//documents/...`) returns the same shape.

## Why it matters

An agent that reads `error.code` gets `undefined`, and one that reads `error.message` gets `undefined` too, because `error` is a string here. The output is valid JSON and structurally wrong, which is worse than prose — a consumer has no signal that it should not trust the shape.

## Suggested direction

Validate before forwarding, and fall through to `outputError` otherwise:

```ts
const err = (body as { error?: unknown } | null)?.error;
if (err && typeof err === "object" && typeof (err as { code?: unknown }).code === "string") {
  console.error(JSON.stringify(body, null, 2));
  process.exitCode = 1;
  return;
}
throw new Error(`HTTP ${res.status}`);
```

Six sites share the pattern, so a small shared helper would keep them from drifting.

---
Found by the CLI issue hunter and hand-verified against `b252617d`. Not auto-filed.