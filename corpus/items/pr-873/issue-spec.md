# Sheets: Increase then decrease decimal places pins the cell to 0 decimals

## What happens

**Increase decimal places** followed by **Decrease decimal places** never returns a cell to
how it started. It leaves an explicit `{ dp, nf: "number" }` behind where the cell
previously had no style at all, and on an untouched cell that residue is `dp: 0` — which
pins the cell to zero decimals and silently rounds everything typed into it afterwards.

1. Open a sheet and click **A1** (empty)
2. Click **Increase decimal places** once
3. Click **Decrease decimal places** once — the cell looks untouched
4. Type `12.5` and press Enter

**Expected:** `12.5`, as in any other empty cell.
**Actual:** **`13`**. The stored value is still `12.5`; only the rendering is wrong, which
is what makes it easy to miss.

Side-by-side: do the above in A1, then type `12.5` into an untouched D1. A1 shows `13`,
D1 shows `12.5`.

## The residue is always there — only its visibility changes

The round trip is not symmetric in any case; what varies is whether you can see it:

| cell before the round trip | style after | visible then? |
|---|---|---|
| empty | `{dp: 0, nf: "number"}` | not until you type a decimal — then it rounds |
| `12` (integer) | `{dp: 0, nf: "number"}` | not until you change it to `12.5` — then `13` |
| `12.5` | `{dp: 1, nf: "number"}` | not until you change it to `12.567` — then `12.6` |

So a user who clicks the two buttons and sees nothing change has still had a number format
written into the cell, and finds out later.

This is the same residue shape as #749 (`italic: false`) and #793 (`backgroundColor: ""`) —
a reverse operation storing an explicit value instead of restoring the absence of one — but
unlike those it changes what the user sees.

## Why

Both handlers write an explicit `dp`, and neither can express "unset"
(`packages/sheets/src/view/spreadsheet.ts:534` and `:549`):

```ts
public async increaseDecimals() {
  const { dp, nf } = await this.sheet.getActiveDecimalState();
  const patch: Partial<CellStyle> = { dp: dp + 1 };
  if (!nf || nf === 'plain') {
    patch.nf = 'number';
  }
  await this.sheet.setRangeStyle(patch);
}

public async decreaseDecimals() {
  const { dp, nf } = await this.sheet.getActiveDecimalState();
  const patch: Partial<CellStyle> = { dp: Math.max(0, dp - 1) };
  if (!nf || nf === 'plain') {
    patch.nf = 'number';
  }
  await this.sheet.setRangeStyle(patch);
}
```

The starting state is *inferred* rather than stored. `getActiveDecimalState` derives `dp`
from the value string when nothing is set — `0` for an empty cell or an integer
(`packages/sheets/src/model/worksheet/sheet.ts:4157`):

```ts
if (cell?.v) {
  const dotIndex = cell.v.indexOf('.');
  if (dotIndex >= 0) {
    return { dp: cell.v.length - dotIndex - 1, nf: style?.nf };
  }
}
return { dp: 0, nf: style?.nf };
```

That inferred value is never written back, so "no explicit dp" and "dp equal to the
inferred one" are the same thing on the way *in* and different on the way *out*. Walking
the empty-cell case:

| step | `getActiveDecimalState()` | patch written | style after |
|---|---|---|---|
| start | — | — | *(none)* |
| Increase | `{dp: 0}` | `{dp: 1, nf: "number"}` | `{dp: 1, nf: "number"}` |
| Decrease | `{dp: 1, nf: "number"}` | `{dp: 0, nf: "number"}` | `{dp: 0, nf: "number"}` |

`Math.max(0, dp - 1)` clamps at `0` rather than returning to unset, so once a cell reaches
`{dp: 0}` no sequence of these two buttons gets it back to unstyled.

`nf: "number"` has the same one-way problem: it is set on the way in and never removed, so
a plain-format cell keeps a number format it never asked for.

## Expected

Increase-then-decrease with equal counts should leave the cell rendering exactly as it did
before, and a cell that had no explicit `dp`/`nf` should end with none.

<sub>Found by the UI issue hunter (`docs/design/hunter-usage.md`) and hand-verified against
the source before filing.</sub>