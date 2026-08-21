# Docs: turning an inline style off stores an explicit false, so the run never re-merges

## Summary

Toggling a boolean inline style **on and then off** on the same selection does not restore the document. The run stays permanently split, and the fragment keeps an explicit `italic: false` (or `bold: false` / `underline: false`) rather than having the key removed.

Rendering is unaffected today — this is a model/structure issue, with one user-visible consequence noted at the bottom.

## Minimal reproduction

1. In a doc, type `abcdef`.
2. Read the runs — the paragraph is a **single run**.
3. Put the caret before `c` and Shift+ArrowRight twice to select `cd`.
4. Click **Italic** → `cd` splits out as `italic: true`. ✅ correct
5. Click **Italic** again → expected: back to one run. Actual:

```jsonc
// baseline (step 2)
[{ "text": "abcdefSmall ", "fontSize": 11 }]

// after italic ON (step 4)
[{ "text": "ab", "fontSize": 11 },
 { "text": "cd", "fontSize": 11, "italic": true },
 { "text": "efSmall ", "fontSize": 11 }]

// after italic OFF (step 5) — still three runs, and a dead flag
[{ "text": "ab", "fontSize": 11 },
 { "text": "cd", "fontSize": 11, "italic": false },   // ← was absent before
 { "text": "efSmall ", "fontSize": 11 }]
```

Every on/off cycle on a fresh sub-range adds another permanent fragment.

## Why the runs never re-merge

`normalizeInlines` merges adjacent runs only when `inlineStylesEqual` says their styles match, and that comparison is strict:

```ts
// packages/docs/src/model/types.ts
export function inlineStylesEqual(a: InlineStyle, b: InlineStyle): boolean {
  return (
    a.bold === b.bold &&
    a.italic === b.italic &&
    ...
```

```ts
// packages/docs/src/store/block-helpers.ts
if (last && inlineStylesEqual(last.style, inline.style)) {
  last.text += inline.text;
} else {
  merged.push({ text: inline.text, style: { ...inline.style } });
}
```

`false === undefined` is `false`, so a run carrying `italic: false` can never merge with an identical-looking neighbour that simply lacks the key. Clearing the key instead of writing `false` would let `normalizeInlines` do its job.

## The consequence beyond tidiness

Style resolution layers named-style defaults **underneath** the explicit run style (`{ ...defaults, ...inline.style }`, `packages/docs/src/view/editor.ts:1008`). So a run carrying an explicit `italic: false` will **not** follow a named style that is later redefined to be italic — the dead flag wins.

That is the exact hazard the codebase already documents, a few lines above:

> Without it — the default — the raw explicit style is returned, so pending-style capture and style application never bake a style default into a stored run (**which would break the lazy cascade when the style is later redefined**).
> — `packages/docs/src/view/editor.ts:990-996`

The off-toggle appears to bake exactly such a value in, by a different route.

## Note on severity

Reported at `major` by the hunter; an automated verifier refused it at that level on the grounds that rendering is unaffected. Both readings seem defensible, so severity is left to a maintainer — the mechanism above is what has been verified.

---
Found by the UI issue hunter and hand-verified against `357475f00`, reduced from a 50-action session to the 5 steps above. Related to #715 (also `getSelectionStyle`-adjacent, but a distinct mechanism: that one is the toggle reading the wrong run; this one is what the write leaves behind). Not auto-filed.