# Docs: B/I/U/S toggles read the caret, not the selection, so a backward selection toggles the wrong way

## Summary

The docs toolbar's **Bold, Italic, Underline and Strikethrough** buttons decide whether to *add* or *remove* a style by reading `editor.getSelectionStyle()` — which samples a **single caret position**, not the selected range. With a **backward** (right-to-left) selection the caret sits at the range's *start*, so the toggle reads the style of the run **preceding** the selection and inverts the wrong value.

The visible consequence: clear a style from a sub-range inside a styled run using a backward selection, and **that button can never put it back**. Every subsequent click is a silent no-op.

## Minimal reproduction

1. In a doc, type `abcdef`.
2. Select all six characters and click **Bold** → all six are bold.
3. Place the caret after `d`, then **Shift+ArrowLeft twice** to select `cd` *backward*.
4. Click **Bold** → bold is removed from `cd`. ✅ correct
5. Click **Bold** again → **nothing happens.** No number of further clicks restores it.
6. Now reselect the same `cd` *forward* (caret before `c`, Shift+ArrowRight twice) and click **Bold** → **it works.**

Same two characters, same document state, same button. Only the selection direction differs.

## Why

`getSelectionStyleImpl` reads `cursor.position` and matches with `offset <= inlineEnd`, which resolves to the run that *ends* at the caret:

```js
// packages/docs/src/view/editor.ts:998
const block = doc.findBlock(cursor.position.blockId);   // the caret, not the range
for (const inline of block.inlines) {
  const inlineEnd = pos + inline.text.length;
  if (cursor.position.offset <= inlineEnd) return { ...defaults, ...inline.style };
  pos = inlineEnd;
}
```

After step 4 the runs are `"ab"`(0–2) `"cd"`(2–4) `"ef"`(4–6). A backward selection leaves the caret at offset 2, and `2 <= 2` matches **`"ab"`** — still bold. So `toggleBold` computes `{ bold: !true }` and removes bold from a range that is already unbold: a no-op.

Selected forward, the caret lands at offset 4, which matches `"cd"` (unbold), and the toggle correctly adds.

## Scope: all four toggles, not just Bold

`packages/frontend/src/components/text-formatting/text-format-group.tsx:91-113` wires all four through the same shape:

```js
const current = editor.getSelectionStyle();
editor.applyStyle({ bold: !current.bold });        // italic / underline / strikethrough identical
```

Italic can look unaffected, because it only misfires when the *preceding* run also has italic. Set that up — make the text italic, clear italic from `cd` backward, try to restore — and Italic fails identically. Confirmed.

The header/footer toolbar (`packages/frontend/src/app/docs/docs-formatting-toolbar.tsx:387-401`) uses the same pattern and is presumably affected too.

## Suggested direction

`getRangeStyleSummary()` already exists, is range-aware, and reports this case correctly — it returns `bold: false` for the selection throughout step 5, which is exactly the value the toggle needed. Having the toggles consult the range summary (treating `'mixed'` as "apply") rather than the caret style would fix all four at once.

Worth checking separately whether `getSelectionStyle`'s `<=` boundary is right even for a collapsed caret: at a run boundary it returns the preceding run's style, which is also what a user typing at that boundary would inherit.

---
Found by the UI issue hunter and hand-verified against `b4268f077` — mechanism confirmed by source reading plus three empirical predictions (backward fails, forward succeeds, Italic fails identically under the matching setup). Not auto-filed.