# Pasting a table into a table cell isn't nested

### What happened:

Pasting a table while the caret is **inside a table cell** does not nest the
table into that cell. The pasted table does not appear in the clicked cell —
the paste has no visible effect.

Creating a nested table with the **Insert Table** command works correctly (the
new table nests inside the cell) — only the **paste** path is affected. So the
in-cell nesting capability exists; paste just doesn't use it.

### What you expected to happen:

Pasting a table into a table cell should **nest it into that cell**, matching:

- what the **Insert Table** command already does when the caret is inside a cell, and
- Google Docs / Word behavior.

The pasted table should appear inside the clicked cell, with the caret landing
in the first cell of the pasted table.

### How to reproduce it (as minimally and precisely as possible):

1. Open a document on https://wafflebase.io/
2. Insert a table (call it the outer table)
3. Somewhere else, insert another table, select it, and copy it (Cmd + C)
4. Click into a cell of the outer table (caret is now inside the cell)
5. Paste (Cmd + V)
6. **Expected:** the copied table nests inside the clicked cell
7. **Actual:** nothing happens — the table is not inserted into the cell

### Anything else we need to know?:

- Localized to the paste path. `insertBlocks` in
  `packages/docs/src/view/text-editor.ts` handles a single pasted `table` block
  with the **body-only** `getBlockIndex` + `insertBlockAt`. These don't account
  for the caret being inside a cell — `EditContext` is only
  `'body' | 'header' | 'footer'` (no cell context), and `getContextBlocks()`
  returns only body/header/footer blocks. So a cell target isn't resolved and
  the table isn't inserted into the cell.
- By contrast, the `insertTable` command already detects the cell via
  `layout.blockParentMap` and routes to `insertTableInCell`, so nesting works
  there. The fix is to apply the same cell detection in the paste path
  (`insertBlocks`).
- This is **distinct from #333**, which is about input being misrouted *after*
  pasting an already-nested table into the body. This issue is that pasting a
  table into a cell doesn't nest at all.
- AI assistance: root-cause analysis and phrasing with Claude Code (Claude Opus
  4.8). I observed the behavior myself.

### Environment:

- Operating system: macOS Sonoma 14.6
- Browser and version: Chrome (arm64)