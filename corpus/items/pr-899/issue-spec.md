# Slides: keyboard shortcuts stop reaching the canvas after any toolbar click

## What happens

After clicking **any** toolbar control, keyboard shortcuts stop reaching the slide canvas.
The element stays selected and looks selected, but arrow keys, `Delete` and the z-order
shortcuts all do nothing until you click the canvas again.

1. Click an element on the slide — arrow keys nudge it, as expected
2. Click **Arrange**, then press **Escape** (take no action at all)
3. Press an arrow key

Nothing moves. `Delete` does nothing. `Cmd+ArrowUp` does nothing. The selection is
untouched, so there is no visual cue that anything has changed — the element is still
highlighted and still reported as selected.

Clicking the element again restores everything.

## Repro

Driving the real editor and toolbar, reading the element's stored `x` before and after each
arrow press:

```
baseline                            x 1560 -> 1561   nudge WORKS
after Arrange menu, open+Escape     x 1561 -> 1561   nudge DEAD
after re-clicking the element       x 1562 -> 1563   nudge WORKS
```

It is not specific to `Arrange`, and not specific to menus — a plain action button does it
too:

| after… | canvas keyboard |
|---|---|
| `Arrange` menu, open + Escape | **dead** |
| `Border weight` menu, open + Escape | **dead** |
| `Format painter` (a plain button) | **dead** |

`Delete` is affected identically — with one element selected, the slide still reports 4
elements after pressing it.

## Why

The shortcut rules are gated on `isEditableTarget`, which treats a focused button as "not
the slide canvas" (`packages/slides/src/view/editor/interactions/keyboard.ts:882`):

```ts
if (tag === 'BUTTON') return true;
```

Radix returns focus to the **trigger** when a dropdown closes, and a plain button keeps
focus after a click. So from then on every `keydown` has `e.target` set to that toolbar
button, `isEditableTarget` answers `true`, and every rule in `buildKeyRules` is skipped.

The gate itself is deliberate and its reasoning is sound — the comment above it explains
that without it, `Enter` on a focused toolbar button would enter text-edit mode instead of
activating the button, and `Tab` inside the shortcuts dialog would be hijacked by the
slide's Tab-cycle rule. So the fix is not to remove the gate. The gap is that nothing
returns focus to the canvas once a toolbar interaction is finished, so the gate stays
engaged indefinitely rather than for the duration of the interaction.

## Expected

After a toolbar control has been used and any menu it opened is dismissed, the canvas
should be the keyboard target again — the selected element should respond to arrow keys,
`Delete` and the z-order shortcuts without needing another click.

## Note

`packages/frontend/src/app/board/is-editable-target.ts:27` carries an identical
`tag === "BUTTON"` branch for the board surface. I have not tested board, so this is a
pointer rather than a claim.

<sub>Found by the UI issue hunter (`docs/design/hunter-usage.md`) on the slides surface's
first run. It SPLIT the verifier panel — one verifier confirmed it and verifier 0 "was not
confident", so the run reported nothing and the drop table flagged it. Hand-verified
afterwards, which also widened it: the candidate blamed the Arrange menu, and it is in fact
any toolbar control.</sub>