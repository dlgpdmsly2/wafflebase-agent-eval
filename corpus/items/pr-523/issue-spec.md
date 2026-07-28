# Undo restores the caret but not the selection range

### What happened:

In the document editor, undo restores the **caret position** but not the **selection range** that was active before the edit. After undoing, only the caret is placed; any previously selected range is lost.

The history snapshot records `activeCursorPos` (the caret) but does not record `activeSelection` (the range), so on undo there is nothing to restore the selection from.

### What you expected to happen:

Undo should restore both the caret and the selection range that were active at that history step, matching Google Docs / typical editors (undo brings back the prior selection, not just a collapsed caret).

### How to reproduce it (as minimally and precisely as possible):

1. Open a document on https://wafflebase.io/
2. Type some text, then select a range of it
3. Perform an edit that replaces/transforms the selection (e.g. type over it, or apply formatting)
4. Press undo (Cmd/Ctrl+Z)
5. Expected: the previous selection range is restored
6. Actual: only the caret is restored; the selection range is gone

### Anything else we need to know?:

- Localized to the docs history path in `yorkie-doc-store.ts`: the undo entry stores `activeCursorPos` only, never `activeSelection`.
- AI assistance: drafted with help from Claude Code (repro phrasing and code-path hints). I reproduced the issue myself and confirmed the behavior.

### Environment:

- Operating system: macOS Sonoma 14.6
- Browser and version: Chrome (arm64)