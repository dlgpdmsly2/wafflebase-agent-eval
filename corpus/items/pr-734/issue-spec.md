# Settings shows owner-only controls to non-owner members

### What happened:

As a non-owner member, `/w/:workspaceId/settings` still shows the remove-member trash icon, "Create Invite", and revoke-invite trash icon. All three fail on click (owner-only on the backend).

### What you expected to happen:

These controls should be hidden/disabled for non-owner members, same as the API Keys and Danger Zone sections already are.

### How to reproduce it (as minimally and precisely as possible):

1. As a member (not owner), go to `/w/:workspaceId/settings`.
2. Click any of the above. Action fails. (screenshot attached)

### Anything else we need to know?:

`isOwner` (line 93) already gates API Keys/Danger Zone correctly in `packages/frontend/src/app/workspaces/workspace-settings.tsx`, but isn't applied to the remove-member button (325-337), "Create Invite" (357-365), or "Revoke invite" (397-407).

<img width="1512" height="822" alt="Image" src="https://github.com/user-attachments/assets/3ce16a6b-bdab-46bb-a5a6-12c2bff320ac" />

### Environment:

- Operating system: MacOS 26.5.2
- Browser and version: Chrome 150.0.7871.115