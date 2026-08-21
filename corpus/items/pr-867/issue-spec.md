# Docs: Highlight "None" stores backgroundColor:"" instead of clearing it

## What happens

Clearing a highlight with **None** does not clear it. It writes `backgroundColor: ""`
into the run, so the run keeps a dead value and never merges back into its
neighbours.

1. Select part of a paragraph and apply a highlight colour
2. Select the same range, open **Highlight color**, choose **None**

The highlight stops rendering, so it looks fixed. The stored document does not go
back to what it was.

## Repro

Driving the real store helper with exactly what the toolbar calls:

```ts
const block = { id: 'b1', type: 'paragraph', style: {},
                inlines: [{ text: 'abcdef', style: {} }] };

const on  = applyInlineStyle(block, 2, 4, { backgroundColor: '#FFF176' });
const off = applyInlineStyle(on,    2, 4, { backgroundColor: '' });   // "None"
```

```
AFTER ON : [{"text":"ab",...{}}, {"text":"cd", "backgroundColor":"#FFF176"}, {"text":"ef",...{}}]
AFTER OFF: [{"text":"ab",...{}}, {"text":"cd", "backgroundColor":""},        {"text":"ef",...{}}]
```

Still three runs. `inlineStylesEqual({ backgroundColor: '' }, {})` is `false`.

## Why

**None** passes an empty string rather than clearing the property
(`packages/frontend/src/components/text-formatting/text-format-group.tsx:344`):

```tsx
onReset={() => handleColor("")}
```

which reaches `editor.applyStyle({ backgroundColor: "" })`, and the store merges the
incoming style over the existing one without treating `""` as "remove"
(`packages/docs/src/store/block-helpers.ts:346`):

```ts
newInlines.push({
  text: inline.text.slice(overlapStart, overlapEnd),
  style: { ...inline.style, ...resolvedStyle },
});
```

`normalizeInlines` then cannot re-merge the run, because `inlineStylesEqual` compares
colours with `storedColorsEqual`, and `'' === undefined` is false — an unset
background and one explicitly set to `""` are different values.

## Why it matters beyond the extra run

- The document keeps accumulating runs that differ only by a meaningless field, and
  the split is permanent — no later edit re-merges them.
- `""` is not a colour. Anything reading `backgroundColor` and treating a present
  value as "there is a highlight" sees one, and export paths may emit it.

## Related

Same shape as #749 (turning an inline style off stores an explicit `false`): the
toolbar's "off" writes a falsy value where it should remove the property. Worth
fixing together — both want "clear" to mean `delete`, not "assign something falsy".

The text-colour **Reset** entry is `onReset={() => handleTextColor("")}` on the same
component and takes the same path; #728 already covers the visible half of that one.

## Provenance

Found by the UI hunter (run `colour1`) on the first run after `doc.runs` began
reporting per-run `color`/`backgroundColor`. The panel reproduced it but one verifier
was not confident, so it was dropped rather than reported; the mechanism above is
hand-verified.