---
name: tla-state-machine-bugs
description: Find state-machine bugs in a codebase by extracting the implicit state machines hidden in the code, writing them out explicitly, modeling them in TLA+, and model-checking with TLC to get concrete counterexample traces. Use this skill whenever the user asks to find bugs in lifecycle or status logic (background jobs and queues, multi-step workflows, documents and publishing, accounts and permissions, sessions and connections, deployments, sync between systems), mentions race conditions, double-processing, stuck records, impossible states, retries, webhooks, idempotency, TLA+, PlusCal, TLC, formal verification or model checking, or asks to "make the state machine explicit", even if they never say "TLA+".
---

# Finding state-machine bugs with TLA+

Most production bugs in stateful code are transitions nobody thought about: a cancel that races a completion, a retry that runs a job twice, an event that arrives before the record it refers to exists. They live in *implicit* state machines spread across status columns, flags, timestamps, and the code paths that write them. No single file shows the whole machine, so code review rarely catches them.

The loop: make the machine explicit (writing it down surfaces many bugs), model it in TLA+, model-check it, and turn each counterexample into a reproducing test. Come back with *confirmed* bugs, each with a trace, a repro, and a code location, not a pile of speculation.

## Phase 0: Scope

Pick one bounded component with a lifecycle: a status field, a background job or queue consumer, a multi-step workflow, a retry loop, a sync between two systems, a connection or session lifecycle, a deploy or migration process. If the user pointed at a whole repo, list candidates and start with the one involving concurrency, external systems, or side effects that can't be undone. Tell the user which one you picked and why.

Model one machine at a time. Match the effort to the machine: a record with two states and one writer only needs the Phase 1 table. TLA+ pays off with several writers, non-atomic steps, external systems, or retries.

## Phase 1: Extract the explicit state machine

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

## Phase 2: Write the TLA+ spec

Read `references/tla-reference.md` for the spec structure, a worked skeleton, and how to run TLC.

Mirror the **code**, not the design doc. If you model intended behavior, TLC verifies the intent and misses every bug. Where code and intent disagree, model the code and write the intent as an invariant. Split every non-atomic transition into separate actions, model the environment (including expiry and deletion) as actions, and comment each action with its `file:line`.

For data-loss questions, abstract content to a set of edits and merges to set union, then add a history variable of everything the server ever held. The invariant "nothing the server held is lost from it" is one line and finds lost-update bugs in a handful of steps.

## Phase 3: Model-check

**Guard against a spec that proves nothing.** Before trusting a green run, give the model a control: a second `.cfg` that turns off the one guard you believe makes it safe (the lock, the version check, the fix), and confirm that config fails. A control proves the model is sensitive to the mechanism you care about; a generic false invariant (e.g. `status[j1] /= "done"`) only proves states are reachable, so use it when there is no guard to remove. Also check that the distinct-states count isn't suspiciously small.

**Expect the first counterexamples to be modeling errors.** An invariant is often stronger than what the system needs (it ignores a cleanup step, or a state a reader never looks at). Restate it as what the consumer actually relies on, and say so in the modeling notes. That is a correction, not a weakening; weakening is dropping a rule the business depends on.

**Read each counterexample** as a story ("j1 queued, w1 claims it, the watchdog re-queues it, w2 claims it while w1 is still running...") and map every step to its `file:line`.

## Phase 4: Confirm, then fix

A TLC trace is a bug **in the model**. Before calling it a bug in the code:

1. Check each step matches something the real code can do. If not, tighten the model and log it as a modeling correction.
2. Write a failing test that reproduces the trace (interleaved calls, a duplicated webhook, a forced crash between writes). The fastest route is usually the existing test suite with the network or database mock scripted to return the trace's outcomes in order (conflict, then offline, then success).
3. Only then mark it **confirmed**. Anything unreproduced stays **suspected**, with the reason.

Fix the spec first, re-run until it's green *and* the control still fails, then change the code and confirm the repro test passes. Run the neighboring test suites too: a fix that satisfies the model can break behavior the model abstracts away. If the shipped fix differs from the one you modeled, update the model to mirror the shipped one. Never weaken an invariant to get green unless the user agrees the business rule was wrong.

Prefer fixes that make the machine more explicit: one status field instead of overlapping flags; removal as a state (`archived`, `revoked`, `rejected`) instead of a delete; a transition record (a timestamp per state if states are never revisited, otherwise a log table of timestamp, old state, new state, and event); and the stack's state-machine library (e.g. Ruby's `state_machines`, XState) so illegal transitions fail loudly.

## Output

Commit the spec next to the code (e.g. `specs/<machine>/`) with the `.tla`, `.cfg` files (each model's control included), state diagram, a short README covering what's modeled, what's abstracted away, and how to run it, and a script that runs every config and prints holds or violated. Check the repo's own rules for where non-code files may go first. Gitignore TLC's `states/` directory and `*_TTrace_*` files.

Report in this shape:

```markdown
# State-machine review: <component>

## The machine
<state diagram + transition table, condensed>

## Findings
### 1. <one-line title> (confirmed | suspected)
- Trace: <numbered steps, plain English>
- Code: <file:line for each step>
- Impact: <what goes wrong for users or data>
- Repro: <test name or why it couldn't be reproduced>
- Fix: <proposed change, and whether the spec now passes with it>

## Checked and safe
<each candidate the model refuted, and the guard that makes it safe (with its control result)>

## Leads
<suspicions found while reading but not modeled or reproduced>

## Modeling notes
<abstractions, what is NOT covered, constants, state counts>
```

Rank findings by impact, and say what the model leaves out (other services, DB isolation level, clock skew) so a clean run isn't mistaken for proof the whole system is correct.
