# Docs: applying a block type leaves the Text style control showing the old style

## What happens

Applying a block type leaves the **Text style** control showing the old value. Put
the caret in a paragraph, apply `Heading 2`, and the toolbar button still reads
`Normal text` — and re-opening the menu still shows `Normal text` checked. The block
really did change; only the control is stale.

It corrects itself on the next caret move or any *other* formatting action, which is
what makes it easy to miss: click Bold, or move the caret, and the label jumps to
`Heading 2` with no further block-type change.

## Why

The toolbar re-derives its state from cursor-move callbacks
(`packages/frontend/src/app/docs/docs-formatting-toolbar.tsx:322`):

```ts
refresh();
const unsubscribe = editor.onCursorMove(refresh);
```

Style operations that do not move the caret synthesise one by calling
`notifyStyleApplied()`, which exists for exactly this
(`packages/docs/src/view/editor.ts:1237`, and the note at `:972` — "so toolbar
pickers re-derive"):

```ts
function notifyStyleApplied(): void {
  ...
  fireCursorMoveCallbacks(cursor.position, selRange);
}
```

`setBlockType` is the one that does not (`packages/docs/src/view/editor.ts:3034`):

```ts
setBlockType(type: BlockType, opts?: {...}) {
  docStore.snapshot();
  doc.setBlockType(cursor.position.blockId, type, opts);
  invalidateLayout();
  render();
},
```

Its siblings all do — `applyBlockStyle` (`:2980`), and the calls at `:3047`,
`:3080`, `:3088`, `:3096`, `:3176`. `setBlockType` is the outlier.

## Both entry points are affected

The keyboard shortcut is **not** a workaround. It does not go through the method
above, but it does not notify either
(`packages/docs/src/view/text-editor.ts:1028`):

```ts
this.saveSnapshot();
...
this.doc.setBlockType(block.id, 'heading', { headingLevel: level });
this.invalidateLayout();
this.requestRender();
```

`text-editor.ts` has no style-notification path at all, so `⌘⌥2` leaves the same
control equally stale. Menu and shortcut behave identically here.

## Suggested fix

Call `notifyStyleApplied()` at the end of `EditorAPI.setBlockType`, matching its
siblings. The shortcut path needs its own answer, since it bypasses that method.

## Provenance

Found by the UI hunter (run `colour1`) as a `dom.snapshot not-equals` prediction on
ground A: the style menu's accessibility snapshot was byte-identical before and
after applying `Heading 2`, while `doc.blockTypes` confirmed the block had changed.