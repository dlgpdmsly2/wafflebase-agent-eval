# CLI: `status` and `ctx list` print prose and ignore `--format` entirely

## Summary

`docs/design/cli.md:687-689` (§8.1 Structured Output) promises:

> All output is JSON by default. Errors are also JSON so agents can parse success and failure uniformly.

`status` and `ctx list` emit **bare English prose** on stdout, ignore the documented global `--format` flag entirely, and do not reject an invalid value for it.

## Reproduction

```console
$ wafflebase status
Not logged in. Run `wafflebase login`.

$ wafflebase status --format json          # byte-identical
Not logged in. Run `wafflebase login`.

$ wafflebase status --format bogus          # not rejected either
Not logged in. Run `wafflebase login`.

$ wafflebase ctx list --format json         # same
Not logged in. Run `wafflebase login`.
```

All exit 0. No backend or auth required.

## Cause

`packages/cli/src/commands/status.ts` never reads the format option — its action takes no `opts` at all, and every branch is a bare `console.log` of a human sentence:

```ts
.action(() => {
  const session = loadSession();
  if (!session) {
    console.log('Not logged in. Run `wafflebase login`.');
    return;
  }
  console.log(`Logged in as ${session.user.username} (${session.user.email})`);
```

It never calls the `output()` formatter that every structured command routes through. `ctx list` behaves the same way.

## Why it matters

`status` is precisely the command an agent calls **first**, to decide whether to prompt for login. It is listed in `wafflebase schema` as a first-class command (`{"name": "status", "safety": "read-only"}`) and in the safety table at `docs/design/cli.md:838`. An agent following §8.1 and calling `JSON.parse` on its stdout throws instead of branching — and the failure is at the very start of any session, before anything else can be attempted.

The silently-accepted `--format bogus` compounds it: there is no signal at all that the flag is unsupported here.

## Suggested direction

Route both commands through `output()` so `--format` is honoured, and reject unknown format values rather than ignoring them. If prose is deliberately wanted for humans, that is what a non-JSON `--format` is for — the default should still satisfy the documented contract.

## Environment

`main` at `dc538024a`, built from source (`pnpm cli build`).

## Provenance

Found by the CLI issue-hunting harness (#600/#602) on its first live run — it noticed `status` and `status --format json` returning byte-identical output — then verified by hand. This is ground-truth defect #9 from the original hunting design, now confirmed empirically. The `--format bogus` and `ctx list` observations are mine, from checking the blast radius.