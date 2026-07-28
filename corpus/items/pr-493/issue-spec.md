# Document type filter buttons show their label with a noticeable hover delay

<!-- Please only use this template for submitting common issues -->

### Description:

On the Documents list page, the type filter buttons (Sheets / Docs / Note /
Slides / PDF icon buttons) use the native HTML `title` attribute to show
their label on hover, instead of the app's own `Tooltip` component
(`components/ui/tooltip.tsx`):

```tsx
// packages/frontend/src/app/documents/document-list.tsx (~line 622-640)
<Button
  ...
  title={label}
>
  <Icon className={`h-4 w-4 ${color}`} />
</Button>
```

Native `title` tooltips have a browser/OS-dependent hover delay (roughly
500ms-1.5s) before they appear. This is inconsistent with other tooltips on
the same page — e.g. the author avatar tooltip further down in
`document-list.tsx` and in `document-presence-avatars.tsx` — which use the
shared `Tooltip` / `TooltipContent` components backed by a
`TooltipProvider` with `delayDuration = 0`, so they appear immediately.

### Why:

Because the type filter controls are icon-only buttons, the label tooltip
is the only way to confirm what each icon means. The inconsistent, noticeable
delay makes these buttons feel unresponsive compared to the rest of the page
and slows down discovery for new users. Switching to the shared `Tooltip`
component would fix the delay and keep tooltip behavior consistent across
the page.