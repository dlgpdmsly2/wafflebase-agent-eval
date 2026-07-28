# Add support for collapsible sections (<details>/<summary>) in Notes

<!-- Please only use this template for submitting common issues -->

### Description:
We need to implement a foldout (collapsible) feature within the Notes, similar to the behavior supported by standard GitHub Markdown and MDN Web Docs.
Users should be able to wrap content in `<details>` and `<summary>` tags to hide and reveal details.

<details>
<summary>Example</summary>
Like this!
</details>

Details would be different in this project, but usually
- By default, the markdown inside the `<details>` tag should be collapsed.
- When the user clicks the `<summary>` label, the details should expand.
- If the `<details open>` attribute is used, the section should be rendered as expanded by default.

### Why:
As notes grow longer or include large blocks of code, the readability significantly decreases. Allowing users to fold/collapse supplementary information or long code snippets will keep the document clean and organized. It provides a much better user experience by letting users focus on the core content and only expand details when necessary.