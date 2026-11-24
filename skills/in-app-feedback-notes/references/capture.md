# Capture (client)

The flow is a small state machine: `null → composing ⇄ picking → capturing → composing → null`. Keep the stage in app state so a keyboard shortcut, a menu item, and an error toast's "Report" button can all start it.

```
Send feedback ──► composing (type first)
                    │  "Point at something"      "Add screenshot"
                    ▼                                  │
                 picking ── pick(el | whole screen) ──►capturing ──► composing
                    │ Esc: back to composing if a draft exists, else close
```

Keep the typed draft (type, intent, text) in a ref outside the dialog, because picking and capturing unmount the dialog.

## Describing the target

`describeTarget(el)` returns what the server stores and what the read-back needs:

```ts
type Target = {
  label: string        // "Button "More tools" in Text card "Plan""
  selector: string     // shortest selector unique in the document
  path: string         // readable ancestry, no nth-of-type
  tag: string
  testId?: string; role?: string; ariaLabel?: string; text?: string
  entityId?: string    // the app's own id for the object the element belongs to
  rect: { x: number; y: number; width: number; height: number }
}
```

**Snap to meaningful.** A click lands on a leaf. Walk up with `el.closest(MEANINGFUL)` where `MEANINGFUL` is `button, a[href], input, textarea, select, label, img, canvas, iframe, h1, h2, h3, h4, li, p, [role], [data-testid], [contenteditable="true"], [data-<entity>-id]`.

**Selector segments**, best first, per element:

1. `tag[data-<entity>-id="…"]` (before test id: test ids repeat across windows of the same kind)
2. `tag[data-testid="…"]`
3. `tag#id` when the id has no long digit runs (generated ids change every render)
4. `tag[aria-label="…"]`
5. `tag.class1.class2` with state classes (`is-*`, `has-*`) and hashed classes (`/[0-9a-f]{6,}/`) removed, plus `:nth-of-type(n)` only when siblings share the tag

Build from the element upward. Each time the current segment is an **anchor** (entity id, test id, stable id, or aria-label) or the chain hits 8 segments, test `document.querySelectorAll(selector)` for exactly one match equal to the element; return the first that passes. Stopping at an accessible name keeps a labeled control's selector free of `nth-of-type` steps that break when nearby chrome changes.

**Path** is the same segments without `nth-of-type`, at most 8, outermost first. It is for humans and for the last-resort match.

**Label** is `<Noun> "<aria-label or short text>"` plus ` in <Entity kind> "<title>"` when inside an app object. Noun comes from `role`, else tag (`Button`, `Link`, `Field`, `Paragraph`, `Heading`, `List item`, `Area`). Cap text at 40 characters for the label and 140 for `text`; never read input values.

Clip every string before sending (label 300, selector 1000, path 2000).

**Stable anchors pay back fast.** If the chrome people point at has no test ids, add them as part of this work. Notes on unlabeled chrome go stray on the next redesign.

## The picker

```ts
const BLOCKED = ['pointerdown','pointerup','mousedown','mouseup','click','dblclick','contextmenu']
window.addEventListener('pointermove', onMove, true)
for (const t of BLOCKED) window.addEventListener(t, onBlocked, true)
window.addEventListener('keydown', onKey, true)
```

- Every listener returns early for events inside the feedback UI (`target.closest('.feedback-ui')`); otherwise `preventDefault()` and `stopPropagation()`, so editors, drag handles and links underneath never react.
- Mouse: hover highlights (throttled to one `elementFromPoint` per frame) with a floating label; click picks.
- Touch (`pointerType !== 'mouse'`, or `(pointer: coarse)` at start): ignore `pointermove` with `buttons === 0` (iOS replays stale positions between touches). A touch selects, dragging moves the selection, and a banner offers **Bigger** (parent), **Smaller** (back down a trail of previous elements), **Use this**, **Cancel**.
- Keys: Esc cancels, Enter picks the highlighted element, ArrowUp / ArrowDown = Bigger / Smaller.
- A **Whole screen** button picks `null`.
- Add a class on `<html>` while picking to set `cursor: crosshair` and disable text selection.
- Announce the highlighted label in a polite live region for screen readers.

## Screenshot

- Lazy-load the DOM-to-image library on first use so it never weighs on first paint (snapdom, html-to-image, or similar).
- Wait for `document.fonts.ready`, then two `requestAnimationFrame`s after the picker unmounts.
- Exclude every element with the feedback UI class. Clip to the viewport. Use `dpr = min(devicePixelRatio, 2)` and an explicit background color.
- Time out (about 12s) and continue without an image rather than blocking the note. Cross-origin iframes render as placeholders; say so if it matters.
- **Redaction**: in the dialog's preview, a "Hide" mode lets the person drag black rectangles. Convert from preview pixels to canvas pixels (`canvas.width / window.innerWidth`).
- **Compose** on a copy: draw the capture, fill redactions black, stroke an outline around the target rect.
- **Crop** around the target with ~48 CSS px of padding. The crop is what agents look at first; the full screenshot is optional context.
- **Encode** JPEG under a byte budget: try quality 0.85, 0.7, 0.55, then halve dimensions and repeat (4 rounds). Notes committed to git should stay small, around 400 KB for the crop.
- With no target, the whole (redacted) screen shrunk to the same budget is the image.

## Context snapshot

Freeze it when the dialog opens, so it describes the moment of the problem, not the moment of sending:

- route (redacted), build sha and env, viewport, dpr, user agent, online state
- app-level summary: which document, counts, focused and selected objects (ids, kinds, short titles). **Never bodies or user content.**
- sync and collaboration status, pending writes
- last ~20 app diagnostics and last ~30 console errors and warnings
- the trail (below)

Console buffer: wrap `console.error` and `console.warn` once, call the original first (so DevTools call-site links are unchanged), stringify safely (circular refs, Errors with stacks), cap each message at 1000 characters and the buffer at 50, and guard re-entry.

Redact before anything leaves the device:

```ts
text
  .replace(JWT, '[jwt]')                        // eyJ…\.…\.…
  .replace(/(bearer)\s+\S+/gi, '$1 [redacted]')
  .replace(/([?&](token|key|secret|code|access_token|refresh_token)=)[^&\s]+/gi, '$1[redacted]')
  .replace(PROVIDER_KEYS, '[redacted]')          // the app's own API key shapes
  .replace(EMAIL, '[email]')
route.split(/[?#]/)[0].replace(/^\/share\/[^/]+/, '/share/:token')  // capability segments
```

## Trail: the last 10 seconds

Install once at startup, next to the console wrapper. Passive capture-phase
listeners on `window`:

- `pointerdown/move/up/cancel`: a press that moved under ~12px is a click, tap or
  pencil tap (from `pointerType`); farther is a drag (record the distance). A second
  pointer down while one is held is one `pinch`; swallow the ups that follow. A
  cancelled touch that never moved is the browser scrolling, not a press.
- `wheel`: fold events less than ~400ms apart into one `scroll` (or `zoom` when
  `ctrlKey`, which is a trackpad pinch) with a step count.
- page changes: wrap `history.pushState`/`replaceState` and listen to `popstate`;
  record the redacted route only when it changed.
- `error` and `unhandledrejection`, redacted; merge the console buffer at snapshot
  time.

Name a press by the nearest control's `data-testid`, accessible name (`aria-label`
or `title`) or role, and the kind of card or panel it sits in. Never read
`textContent` and never listen to key events: the trail goes into a repo.
Skip events inside the feedback UI. Keep a capped ring (2x window, ~80 events).

Freeze it when the feedback flow starts (stage goes from none to picking or
composing), not when the dialog mounts: pointing takes seconds and would push out
what happened. Snapshot = events in `[frozenAt - 10s, frozenAt]`, oldest first, as
offsets in ms (`ms <= 0`), at most 40. Send it with the note; the server renders
it as `## Trail`, one line each: `- -2.4s pinch [canvas] in text card: 3 steps`.

For repo-committed notes, the rest of the context snapshot usually stays out of the file (it goes to the issue-tracker sink, if there is one); `build`, `route` and the target are enough for an agent to start.

## Outbox

The dialog closes the moment the person presses Save. Everything after happens in the background:

1. Encode the images.
2. Put `{ id, ownerId, input }` in a device outbox (IndexedDB, or localStorage for small notes; on quota errors retry without the image).
3. Show the note immediately as a draft with the `saving` signal.
4. Flush oldest first. On a transient failure (offline, 401 sign-out mid-flight, 408, 429, 5xx) stop and keep everything. On a permanent refusal (other 4xx), drop that note and say why in a toast. After ~8 attempts drop it too, so one bad note cannot wedge the queue.
5. Retry on the `online` event, on sign-in, and on each refresh.

Key the outbox by account: on a shared device, notes from one account must neither show for nor be sent as another. Clear the in-memory list when the signed-in account changes.

If an error toast offers "Report", start at `capturing` with the whole screen and attach the error (message, status, request id), skipping the picker.
