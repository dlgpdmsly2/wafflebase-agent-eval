# Analytics "Details" link 403s for non-manager members

### What happened:

Any workspace member sees a "Details" link for every document in Analytics, even ones they don't own/manage. Clicking it shows a generic "Failed to load analytics." error (403 under the hood).

### What you expected to happen:

The link should be hidden/disabled for documents the viewer can't manage, or the error should say it's a permission issue.

### How to reproduce it (as minimally and precisely as possible):

1. As a member (not owner/author), go to `/w/:workspaceId/analytics`.
2. Click "Details" on a document someone else authored.
3. See the error. (screenshot attached)

### Anything else we need to know?:

List endpoint allows any member; detail endpoint requires `isDocumentManager` (owner or author): `packages/backend/src/analytics/analytics.controller.ts:170-174` vs `:220-222`.
Frontend maps the 403 to a generic message:`packages/frontend/src/app/analytics/document-analytics.tsx:54`.

<img width="3024" height="1508" alt="Image" src="https://github.com/user-attachments/assets/3ff68809-ee9a-4d62-b0af-c6e634d14775" />

<img width="3024" height="1486" alt="Image" src="https://github.com/user-attachments/assets/fa03fa0e-c72f-4180-ac93-7c71cbc3d9dc" />

### Environment:

- Operating system: MacOS 26.5.2
- Browser and version: Chrome 150.0.7871.115