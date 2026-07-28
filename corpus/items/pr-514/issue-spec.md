# No underline is drawn on composing (uncommitted) IME text

### What happened:

While composing text with an IME (e.g. Korean), the uncommitted composition text is drawn with **no underline**. The standard IME convention is to underline composing text to indicate it is not yet committed; the docs editor draws the composing glyphs plainly, so there is no visual cue that the text is mid-composition.

### What you expected to happen:

Uncommitted (composing) IME text should be rendered with an underline, removed when the composition commits — matching native text fields and Google Docs.

### How to reproduce it (as minimally and precisely as possible):

1. Open a document on https://wafflebase.io/
2. Switch to a Korean (or other IME) input
3. Start typing a syllable so the IME is mid-composition (before commit)
4. Expected: the composing text shows an underline
5. Actual: the composing text has no underline

### Anything else we need to know?:

- Related: #318. That issue moves composition text onto a **view-local temporary render path** (composing text is currently drawn because it is written to the model). The composing underline is cleanest to implement on top of that new render path, so this is best done after / together with #318 rather than added to the soon-to-be-removed model-drawn path.
- AI assistance: drafted with help from Claude Code (repro phrasing and code-path hints). I reproduced the issue myself and confirmed the behavior.

### Environment:

- Operating system: macOS Sonoma 14.6
- Browser and version: Chrome (arm64)