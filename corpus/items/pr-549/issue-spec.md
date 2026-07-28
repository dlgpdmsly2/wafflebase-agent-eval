# List item text turns into H2 if the nested item is empty

<!-- Please use this template while reporting a bug and provide as much info as possible. Not doing so may result in your bug not being addressed in a timely manner. Thanks!
-->

### What happened:
When a list item is followed by an empty nested list item (bullet), the text of the parent list item is incorrectly wrapped in an `<h2>` tag. The parent item still displays the bullet point indicator, but its text inadvertently takes on Header 2 styling.

### What you expected to happen:
The parent list item should retain standard body text styling, and the empty nested bullet should simply render as an empty child item without altering the parent's HTML structure or styling.

### How to reproduce it (as minimally and precisely as possible):
1. Create a list item with some text (e.g., - 1).
2. Move to the next line and create an indented, empty nested list item (e.g.,  -).
3. The raw input should look exactly like this:
```
- 1
  -
```
<img width="621" height="92" alt="Image" src="https://github.com/user-attachments/assets/42a01333-3d11-49b7-96df-c2882ce12148" />

### Anything else we need to know?:
This appears to be a Markdown parsing bug. It seems the parser is mistakenly interpreting the hyphen (-) of the indented, empty bullet point as a Setext heading underline for the line immediately above it.

Workaround:
If an empty line is inserted between the parent item and the nested item, the issue does not occur and it renders normally. For example:
```
- 1

  - 
```
This workaround further suggests that the issue is tied to Setext heading parsing logic, as the blank line breaks the heading syntax condition.

### Environment:

- Operating system: Microsoft Windows 25H2
- Browser and version: Chrome 150.0.7871.129