# CLI: --dry-run is silently ignored by docs list/get and sheets tabs list/cells get

## Summary

Four commands accept `--dry-run` and then perform the request anyway. Every sibling read command honours it, so this reads as four missed wirings rather than a deliberate scope.

**No data-loss risk** — every write and destructive command honours the flag correctly (verified below). The cost is that `--dry-run` cannot be trusted as a uniform "show me what you would do" for inspection or for agent pre-flight.

## Documented contract

`docs/design/cli.md:707-708` (§8.2 Dry-Run):

> `--dry-run` validates inputs, resolves the target API endpoint, and prints the request that would be sent — without executing it.

## Affected

| command | `--dry-run` |
|---|---|
| `docs list` | **ignored** |
| `docs get <id>` | **ignored** |
| `sheets tabs list <id>` | **ignored** |
| `sheets cells get <id> <tab>` | **ignored** |
| `docs content`, `slides content`, `notes content` | honoured |
| `docs create` / `rename` / `delete` | honoured |
| `sheets cells set` / `batch`, `sheets import` | honoured |

`printDryRun` has 19 call sites (`packages/cli/src/commands/{docs,slides,notes,cells,sheets-import}.ts`), including read-only `GET .../content` paths — the four above simply never call it.

## Reproduction

Point the CLI at a port where nothing listens, so a sent request is provable and nothing can be affected:

```
$ export WAFFLEBASE_SERVER=http://127.0.0.1:1 WAFFLEBASE_API_KEY=wfb_dummy WAFFLEBASE_WORKSPACE=ws

$ wafflebase docs list --dry-run
{ "error": { "code": "ERROR", "message": "fetch failed" } }      # exit 1
```

`fetch failed` only happens if the request was actually attempted — a dry-run must never reach the network. Same for `docs get d1 --dry-run`, `sheets tabs list d1 --dry-run`, `sheets cells get d1 t1 --dry-run`.

Contrast a command that honours it:

```
$ wafflebase docs rename d1 newname --dry-run
{
  "dry_run": true,
  "method": "PATCH",
  "url": "http://127.0.0.1:1/api/v1/workspaces/ws/documents/d1",
  ...
}
```

## Why it matters

An agent using `--dry-run` to check what a command would do gets a real request instead — burning rate limit, emitting real access logs, and returning a live payload it was told it would not fetch. Because the flag *is* honoured everywhere else, a caller has no way to know these four are exceptions.

## Suggested direction

Add the same `if (opts.dryRun) { printDryRun(getConfig(opts), 'GET', <path>); return; }` guard these commands' siblings already use, before the client call in each of the four handlers.

---
Found by the CLI issue hunter and hand-verified against `b70302dd1`. Not auto-filed.