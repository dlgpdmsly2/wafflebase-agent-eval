# Migrate WaffleNotes Undo/Redo to Yorkie for multi-user undo

## Summary

Migrate WaffleNotes (`"note"` document type, `@wafflebase/notes`) Undo/Redo from **CodeMirror local history** to **Yorkie-native undo (`doc.history`)**. The goal is **multi-user Undo/Redo** — so that undo carries CRDT semantics and preserves peers' concurrent edits in a collaborative session.

Slides (`slides-native-undo.md`), Docs, and Sheets have already migrated to Yorkie-native undo, so we reuse that established pattern.

## Current State

- Undo/Redo is handled by CodeMirror 6's built-in history (`basicSetup`). `NoteEditorAPI.undo()/redo()` delegate to CM's `undo(view)/redo(view)` — `packages/notes/src/view/editor.ts:347`
- Remote changes are excluded from CM history via `Transaction.addToHistory.of(false)` — `packages/notes/src/view/note-sync.ts:37`. So **undo is purely local** and fully decoupled from Yorkie.
- A dormant forward-compat path exists: `yorkie-note-store.ts:37` has `isUndoRedo = source === 'undoredo'` wired but inactive.
- Per the design doc (`docs/design/notes/notes.md`), this was deliberately deferred from P1 to P2.

## Problem (why this is needed)

During collaborative editing, undo only reverts the local CM snapshot and then re-applies that as a forward edit to Yorkie. When a peer edits concurrently, the local undo gets tangled with or overwrites the peer's change, breaking the UX. In a multi-user setting there is no guarantee that undo reverts "only what I just did."

## Target (reuse the Slides pattern)

- **1 batch = 1 Yorkie change = 1 undo unit** — via an ambient-root + `withUpdate()` helper where only the outermost batch opens a `doc.update()`; nested batches short-circuit.
- **Reverse-op based** — apply only the reverse of changed ops rather than restoring a snapshot → preserves peers' concurrent edits.
- **Undo floor** — prevent undoing below the initial seeded state (avoid wiping the doc to empty).
- Delegate to `doc.history.undo()/redo()/canUndo()/canRedo()`.

## Tasks

- [ ] Add a batch API to the `NoteStore` interface (or expose undo/redo at the store level)
- [ ] Apply the ambient-root / `withUpdate()` pattern in `YorkieNoteStore` — currently each `editText()` opens a fresh `doc.update()`
- [ ] Implement `undo()/redo()/canUndo()/canRedo()` in `YorkieNoteStore` (delegating to `doc.history`)
- [ ] Capture an undo floor after seeding (equivalent to `markUndoBaseline()`)
- [ ] Disable CodeMirror history + route Cmd+Z / Cmd+Shift+Z keybindings to `doc.history`
- [ ] Activate the dormant `isUndoRedo` remote path → feed undo results back into CM as a remote change
- [ ] Tests: batch grouping / undo floor / churn regression (reuse the Slides & Docs suites)
- [ ] Add an undo section to `docs/design/notes/notes.md` (or a new `notes-native-undo.md`)

## Prerequisite Check

Spike to verify that `doc.history.undo/redo` actually works for character-level `Text.edit()` on Yorkie 0.7.13. (Tree `editByPath` merge is documented as non-reversible — confirm at the SDK level how `Text` differs.)

## References

- Pattern template: `docs/design/slides/slides-native-undo.md`
- Reference implementations: `packages/frontend/src/app/slides/yorkie-slides-store.ts` (ambient-root / batch / undo), `packages/frontend/src/app/docs/yorkie-doc-store.ts` (undo floor / native undo)