# Read back (client)

Everything renders into one portal at the end of `<body>`, carrying the feedback UI class so it never shows up in screenshots. The layer mounts only when the feature is on for this account (signed in, and gated by a developer toggle during rollout). Team accounts see every note and an always-present tray; outside accounts see only their own notes, and the tray only once they have one.

## Finding the element again

```ts
function resolveAnchor(note: Note, root: ParentNode = document): Element | null {
  const { selector, entityId, testId, path } = note.target
  if (selector) { const hit = safeQuery(root, selector); if (hit) return hit }
  if (entityId) {
    const obj = safeQuery(root, `[data-entity-id="${esc(entityId)}"]`)
    if (obj) return (testId && safeQuery(obj, `[data-testid="${esc(testId)}"]`)) || obj
  }
  if (testId) { const hit = safeQuery(root, `[data-testid="${esc(testId)}"]`); if (hit) return hit }
  return path ? safeQuery(root, path) : null        // safeQuery catches invalid selectors
}
```

Only resolve notes whose `route` matches the current (redacted) route. Null, or an element with a zero-size box, means **stray**: never guess a nearby element.

## Placing marks

A `requestAnimationFrame` loop measures every resolvable note's `getBoundingClientRect()` and calls `setState` only when some box changed. This keeps marks glued to elements through scroll, pan, zoom, transforms and layout shifts without wiring into every one of them. Start it from a ref callback on the layer (not an effect that derives state), stop it on unmount.

A mark is a small button at the element's top-right corner, colored by note type, with an accessible name like `Bug note: <title>`. Clicking toggles the slip.

## Signals

One thin line under each mark (and on each tray row) says what is new, in this priority:

| Signal | When |
|---|---|
| `saving` | the note is still in the device outbox (pulse; static under reduced motion) |
| `done` | status is done |
| `update` | Now is non-empty and `updated` is newer than the stamp this device saved when it last opened the note |
| `working` / `review` | status |
| `quiet` | nothing new |

Store "seen" stamps per device (`{ [id]: updated }` in localStorage), written when a slip opens and when this device's own write returns.

## Tray

A small fixed button in a corner reading **Feedback N** (open notes), with a dot when any note has `update`. Expanded, it groups:

- **On this page**: open notes whose element resolved
- **Stray on this page**: same route, element not found (show the stored label)
- **Elsewhere**: other routes (show the route; clicking can navigate)
- **Recently done**

Empty state: "No notes yet. Use Send feedback to annotate anything." Show load errors inline.

## Slip

A popover anchored to the mark (or to the tray when stray), placed with the app's menu-placement helper so it stays on screen and avoids hinges or notches. Contents:

- type, status, author, the note text, the image
- **Now**, rendered as markdown
- Inbox entries not yet folded in
- a follow-up box (sends `PATCH { followUp }`, which reopens a done note)
- **Mark done** / **Reopen**
- **History**: link to `https://github.com/<owner>/<repo>/commits/<branch>/<path of the note>`, which is the note's whole conversation as diffs
- `pr` link when present; "Saving…" plus the last send error while in the outbox

Dismiss on Escape and outside click; focus the slip on open without scrolling.

## Polling

Refresh on mount, on `visibilitychange` to visible, and on an interval only while visible: about 60s for team accounts, about 5 minutes for outside accounts. Every refresh flushes this account's outbox first, then fetches, then merges outbox drafts the server has not returned yet. Ignore a response if the signed-in account changed while it was in flight.

## Look

Use the app's existing tokens (accent, type colors, surface, radius) rather than new colors. Keep marks small (a dot or tab of 10 to 14 px) with a 44 px invisible hit area on touch. Respect `prefers-reduced-motion`.
