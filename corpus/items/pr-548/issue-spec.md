# Allow text selection and link interaction in Docs viewer mode

### What happened:

In Docs viewer mode, readers cannot select text by dragging, copy text, or open hyperlinks embedded in the document. This limits basic document review and navigation even though these actions do not modify the document.

### What you expected to happen:

Viewer mode should remain read-only for document editing while still allowing readers to:

- Select text by dragging.
- Copy selected text.
- Click hyperlinks to open their destinations.

### How to reproduce it (as minimally and precisely as possible):

1. Create a Docs document containing regular text and a hyperlink.
2. Open the document through a viewer-role share link.
3. Try to select and copy text.
4. Try to click the hyperlink.
5. Observe that none of these interactions work.

### Anything else we need to know?:

Read-only mode should prevent content changes without disabling non-editing interactions needed to read and use the document.

### Environment:

- Operating system: Not specified
- Browser and version: Not specified