# Docs: un-listing a heading downgrades it to a paragraph, losing the heading level

## What happens

Applying a list to a heading and then removing the list leaves a plain
paragraph. The heading level is destroyed, silently, by an action that looks
reversible.

1. Put the caret in a `Heading 2` block
2. Click **Bulleted list** — the block becomes a list item
3. Click **Bulleted list** again to turn it off

Expected: a `Heading 2` again. Actual: body text.

Google Docs and Word both keep the heading through this round trip (the bullet
is applied *to* the heading), so this is a deviation rather than a house style.

## Why

Two independent pieces, both required.

`setBlockType` clears the heading level unconditionally and only ever restores
it when the new type is `heading` — so the level is gone the moment the list is
applied (`packages/docs/src/store/memory.ts:249`):

```ts
block.type = type;
delete block.headingLevel;
delete block.listKind;
delete block.listLevel;
if (type === 'heading') {
  block.headingLevel = opts?.headingLevel ?? 1;
}
```

`toggleList` then hardcodes `'paragraph'` on the way out, so there is nothing to
restore from even if the level had survived
(`packages/docs/src/view/editor.ts:3100`):

```ts
toggleList(kind: 'ordered' | 'unordered') {
  docStore.snapshot();
  forEachBlockInSelection((block) => {
    if (block.type === 'list-item' && block.listKind === kind) {
      doc.setBlockType(block.id, 'paragraph');
```

Nothing anywhere records the previous block type — there is no `prevType` /
`previousType` / `restoreType` in either file.

## Repro at the model level

Driving the store directly with exactly the two calls `toggleList` makes:

```ts
const block = { id: 'b1', type: 'heading', headingLevel: 2,
                inlines: [{ text: 'Quarterly results', style: {} }], style: {} };
const store = new MemDocStore({ blocks: [block] });

store.setBlockType('b1', 'list-item', { listKind: 'unordered', listLevel: 0 });
store.setBlockType('b1', 'paragraph');

store.findBlock('b1');   // { type: 'paragraph' }  — headingLevel is gone
```

## Scope

This is about the **round trip**, not about how a list item is styled while it
is a list item. `blockStyleId` mapping `list-item` to `normal`
(`packages/docs/src/model/named-styles.ts:97`) is a styling derivation for the
block's current state and is working as intended; it does not speak to what the
block should return to when the list is removed.

Undo still recovers the heading — `toggleList` snapshots first. What is lost is
the round trip through the toolbar, which is the path a user takes when they
change their mind two edits later.