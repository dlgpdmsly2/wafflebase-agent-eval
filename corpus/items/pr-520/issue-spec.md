# Link formatting persists after pressing Enter or Space

### What happened:
After inserting or pasting a hyperlink in Wafflebase Docs, the hyperlink formatting remains active when pressing Enter or Space and continuing to type. As a result, text entered after the hyperlink unintentionally becomes part of the same hyperlink.

### What you expected to happen:
Pressing Enter at the end of a hyperlink should create a new paragraph without hyperlink formatting. After pressing Space at the end of a hyperlink, users should also be able to continue typing normal text outside the hyperlink.

### How to reproduce it (as minimally and precisely as possible):
  1. Open a document(or create new document) in Wafflebase.
  2. Type some text and apply a hyperlink to it, or paste a URL that becomes a hyperlink.
  3. Place the cursor at the end of the hyperlink.
  4. Press Enter or Space.
  5. Continue typing.
  6. Observe that the newly entered text remains part of the previous hyperlink.

### Anything else we need to know?:
The issue occurs because there is no intuitive way to exit the hyperlink formatting using Enter or Space. Users must manually remove the hyperlink formatting before continuing to type normal text.

### Environment:
  - Operating system: macOS Sequoia 15.3.1 (processor: Apple M4 Pro, memory: 48 GB)
  - Browser and version: Arc Version 1.153.1 (82775) / Chrome Version 149.0.7827.201 (Official Build) (arm64)

https://github.com/user-attachments/assets/ab34e023-0e7c-4b35-93c5-b99d36e84d23