# Write path (server)

One authenticated route with three methods. Any serverless platform works; the token never reaches the client.

```
GET    /api/annotations   → { annotations: Note[] }   open + last ~20 done, filtered by visibility
POST   /api/annotations   → 201 { annotation }        create (idempotent on id)
PATCH  /api/annotations   → { annotation }            { id, followUp? , status? }
```

Return 503 with a clear message when the token is missing, so a preview without secrets fails loudly instead of silently dropping notes.

## Request schema

```ts
const create = z.object({
  id: z.string().regex(/^[A-Za-z0-9_-]{8,64}$/),
  type: z.enum(['bug', 'idea', 'question', 'other']),
  intent: z.enum(['decide', 'ask']).optional(),
  note: z.string().trim().min(1).max(MESSAGE_MAX),
  route: z.string().max(500),
  target: targetSchema.nullable(),
  build: z.string().max(80).optional(),
  crop: z.string().max(Math.ceil((CROP_MAX_BYTES * 4) / 3) + 4).nullable(), // base64 JPEG
})
const update = z.object({
  id: idSchema,
  followUp: z.string().trim().min(1).max(MESSAGE_MAX).optional(),
  status: z.enum(['open', 'working', 'review', 'done']).optional(),
}).refine((v) => v.followUp !== undefined || v.status !== undefined)
```

## Committing through the Git Data API

The Contents API makes one commit per file and loses races. Use blobs, a tree, a commit and a non-forced ref update, so the note and its image land together and a concurrent write is detected:

```ts
async function transact(message, build: (read) => Promise<FileChange[] | null>) {
  for (let attempt = 1; attempt <= 3; attempt++) {
    const head = (await gh('GET', `/repos/${o}/${r}/git/ref/heads/${branch}`)).object.sha
    const changes = await build((path) => readBlobText(`${head}:${path}`))   // reads at this head
    if (!changes) return false
    const baseTree = (await gh('GET', `/repos/${o}/${r}/git/commits/${head}`)).tree.sha
    const entries = await Promise.all(changes.map(async (c) =>
      c.delete ? { path: c.path, mode: '100644', type: 'blob', sha: null }
               : { path: c.path, mode: '100644', type: 'blob',
                   sha: (await gh('POST', `/repos/${o}/${r}/git/blobs`,
                         { content: c.text ?? c.base64, encoding: c.text ? 'utf-8' : 'base64' })).sha }))
    const tree = await gh('POST', `/repos/${o}/${r}/git/trees`, { base_tree: baseTree, tree: entries })
    const commit = await gh('POST', `/repos/${o}/${r}/git/commits`, { message, tree: tree.sha, parents: [head] })
    try {
      await gh('PATCH', `/repos/${o}/${r}/git/refs/heads/${branch}`, { sha: commit.sha, force: false })
      return true
    } catch (e) {
      if (e.status !== 422 || attempt === 3) throw e   // 422: branch moved; rebuild on the new head
    }
  }
}
```

`build` runs again on every attempt, so it must decide from what it reads, not from earlier state.

For reads, the GraphQL `object(expression: "<branch>:annotations")` query returns every blob's text in one round trip. For the archive, list names first and fetch only the newest ~20.

Commit messages: `annotate(<type>): <title>` on create, `annotate: follow up on <id>`, `annotate: mark <id> <status>`. A consistent prefix lets CI and deploy skips (see `agent-loop.md`) recognise note-only commits by path, and makes `git log` readable.

## Create

```ts
const by = isTeam(user)
  ? { author: firstName(user), authorId: authorKey(user), trust: 'team' }
  : { author: 'App user', authorId: authorKey(user), trust: 'outside' }
const input = by.trust === 'team' ? parsed : scrub(parsed)
if (by.trust === 'outside' && openNotesBy(by.authorId) >= OUTSIDE_OPEN_LIMIT && !alreadyMine(input.id))
  return 409 'You have N open notes. Mark one done to add another.'
await transact(`annotate(${input.type}): ${title}`, async (read) => {
  const existing = (await read(openPath(id))) ?? (await read(archivePath(id)))
  if (existing !== null) { taken = parse(existing); return null }
  return [{ path: openPath(id), text: serialize(draft(input, by, now)) },
          ...(input.crop ? [{ path: `annotations/media/${id}.jpg`, base64: input.crop }] : [])]
})
if (taken) return taken.authorId === by.authorId ? 200 { annotation: taken } : 409 'That note id is already taken.'
```

- `authorKey(user) = sha256('<app>-annotation:' + user.id).hex.slice(0, 16)`: stable per account, not reversible, safe in a public repo. Each person sees their own notes by matching it.
- `isTeam` is a server-side allow list (emails or user ids in an env var), never a client flag.
- **Scrub** for outside notes: drop `crop` and `intent`; replace `label` with something like `"<testId or tag> in a <kind>"`; drop `text`, `ariaLabel`, titles; blank `selector` and `path` if they quote an aria-label. Opaque ids (entity id, test id) still re-attach the note.

## Update

Inside one `transact`: read the open path, else the archive path; 404 if missing or the caller may not see it. A follow-up appends to Inbox and reopens a done note; a status change sets it. Bump `updated` and `expires`. If the status moved the file across `annotations/` and `archive/`, write the new path and delete the old in the same commit.

## Visibility

`canSee(user, note) = isTeam(user) || note.authorId === authorKey(user)`. Apply it to GET and PATCH.

## Optional second sink: issue tracker

Plain "Send feedback" from users who will never see the repo can go to an issue tracker instead of a file: insert a row (for rate limits and audit), upload the images to private storage, then create an issue with the note quoted first, then reporter, route, build, viewport, the target (label, selector, path, ids, rect), the images, and collapsible JSON blocks for crash, diagnostics, console and context. Rate-limit per account (20 per hour works). Keep the coordinator loop on files; triage the tracker separately.
