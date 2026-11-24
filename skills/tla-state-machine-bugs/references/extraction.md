# Extracting the explicit state machine

Read this for Phase 1 of the skill.

Build this inventory before writing any TLA+. The spec is only as good as it is.

**State variables.** Everything that encodes where an entity is in its life:

- Status enums and `state` columns
- Boolean flags (`is_published`, `archived`, `verified`)
- Nullable timestamps (`published_at`, `completed_at`), where null means "hasn't happened"
- External state: a third-party API's view, another service's copy, a cache, a queue
- **Time:** start/end dates on records valid for a period (leases, invites, tokens, trials, reservations). The state changes when the clock passes, with no code writing anything. Find out what acts on expiry: a cron job, a check at read time, or nothing.
- **Deletion:** a hard delete is a transition into a "gone" state. It erases history and can strand references, queued jobs, and webhooks.

The real state is the product of all of these, so note combinations the code assumes can't happen (`completed_at` set while `status = "queued"`).

**Transitions.** Every code path that writes a state variable:

| Transition | From (guard) | Writes | Trigger | Atomic? | Location |
|---|---|---|---|---|---|
| Claim | status = queued | status:=running, worker:=w | queue poll | no: SELECT, then UPDATE | `jobs/worker.py:88` |

- **Guard:** what the code actually checks, not what it should. A missing check is a finding.
- **Trigger:** user action, webhook, cron, queue message, timeout, admin tool, migration.
- **Atomic?** If the read-check-write isn't in one transaction or lock, it's really several steps that can interleave. Most race bugs come from here. Answer it from the storage code, not the application code: a conditional `UPDATE ... WHERE version = $expected` is a compare-and-set, an advisory or row lock serializes writers, and the isolation level decides what a second statement sees. Note where ids and sequence numbers are drawn: a value drawn at insert but visible only at commit lets a reader's cursor skip it.
- **Reachable?** Before modeling a path, check that production takes it. Feature flags, live or offline modes, and fallbacks for an older schema can make a path dead. Grep the flags and config, and check which migrations are deployed.

**Environment.** Messages delivered twice, out of order, or never; crashes between two writes; two workers at once; a retry after a timeout that actually succeeded.

Draw the machine (Mermaid `stateDiagram-v2` works) and record **candidates**:

- States with no way out, or that nothing can produce
- Guards that are missing or weaker than others into the same state
- Reachable flag combinations no code handles
- Two writers to one field with no coordination
- Expiry nothing enforces; deleted entities something can still reach
- Internal states exposed through an API, or clients able to request illegal transitions

**Check history if it exists.** An audit log, transitions table, event stream, or per-state timestamps is evidence from production. Ask the user before querying anything outside a dev environment. Look for logged transitions missing from your table, states reached with no legal path, and entities stuck in non-terminal states. A hit here beats any model, because it already happened.

**Check git history too.** Search commits and migrations for races already fixed (a lock, a version check, an idempotency key). Model each such guard as a boolean constant, so the report can say "safe because of X, and the model fails without X" instead of "looks safe".

