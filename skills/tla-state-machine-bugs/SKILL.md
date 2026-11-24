---
name: tla-state-machine-bugs
description: Model-check a concurrent or multi-step lifecycle in TLA+ with TLC to find interleaving bugs (lost updates, double processing, stuck records) and trace each counterexample to the code. Use when the user asks for TLA+, PlusCal, TLC, model checking or formal verification, or asks to hunt for races in logic with several writers, non-atomic steps, or retries across systems (sync, queues, workflows) that tests and code reading have not settled. Not for a single race, retry or webhook question a targeted test can answer.
---

# Finding state-machine bugs with TLA+

Some bugs only show up in an interleaving: a cancel that races a completion, a retry that runs a job twice, an event that arrives before the record it refers to exists. They live in *implicit* state machines spread across status columns, flags, timestamps, and the code paths that write them, where no single file shows the whole machine.

The loop: make the machine explicit (writing it down often settles the question on its own), model only the tricky interleavings in TLA+, model-check them, and treat each counterexample as a hypothesis until a test reproduces it in the real code. Come back with *confirmed* bugs, each with a trace, a repro, and a code location, not a pile of speculation.

This skill finds and reports. Changing production code, committing specs, or opening PRs is a separate step: ask the user before doing it, unless they already asked for fixes.

## Phase 0: Scope

Pick one bounded component with a lifecycle: a status field, a background job or queue consumer, a multi-step workflow, a retry loop, a sync between two systems, a connection or session lifecycle, a deploy or migration process. If the user pointed at a whole repo, list candidates and start with the one involving concurrency, external systems, or side effects that can't be undone. Tell the user which one you picked and why.

Model one machine at a time.

**Try the cheap checks first.** Many questions are settled without a model: read the transaction and lock boundaries, check the queue's delivery semantics (at-least-once? ordered per key?), or write one targeted test that forces the suspected interleaving. Do those first. Reach for TLA+ only when the interleavings are too many to reason about by hand: several writers, non-atomic steps, retries across systems. A record with two states and one writer needs the Phase 1 table at most.

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

**A green run is only as good as the model.** No single check proves the model is faithful, so do all of these before trusting one:

- **Fidelity table.** For each action, name the code path it mirrors (`file:line`) and what it simplifies. Then list what the model leaves out (other writers, failure modes, the isolation level, bounds). Read the list as the limits of the result, and put it in the report.
- **Control config.** A second `.cfg` that turns off the guard you believe makes it safe (the lock, the version check, the fix) must fail. That shows the model is sensitive to that guard; it does not show the model is complete.
- **Reachability.** Confirm the behaviors that matter actually occur in the model, e.g. a temporary invariant that says "a conflict never happens" or "two writers never overlap" should fail. A state that is never reached is a sign of an over-constrained `Init` or guard, and so is a suspiciously small distinct-states count.

**Expect the first counterexamples to be modeling errors.** An invariant is often stronger than what the system needs (it ignores a cleanup step, or a state a reader never looks at). Restate it as what the consumer actually relies on, and say so in the modeling notes. That is a correction, not a weakening; weakening is dropping a rule the business depends on.

**Read each counterexample** as a story ("j1 queued, w1 claims it, the watchdog re-queues it, w2 claims it while w1 is still running...") and map every step to its `file:line`.

## Phase 4: Confirm, then (if asked) fix

A TLC trace is a bug **in the model**. Before calling it a bug in the code:

1. Check each step matches something the real code can do. If not, tighten the model and log it as a modeling correction.
2. Write a failing test that reproduces the trace (interleaved calls, a duplicated webhook, a forced crash between writes). The fastest route is usually the existing test suite with the network or database mock scripted to return the trace's outcomes in order (conflict, then offline, then success).
3. Only then mark it **confirmed**. Anything unreproduced stays **suspected**, with the reason.

If the user wants a fix: trying it in the spec first is a cheap experiment to see whether an idea closes the trace, nothing more. A green model of a hypothetical fix says nothing about the code you then write. After changing the code:

1. The repro test passes, and the neighboring test suites still pass (a fix that satisfies the model can break behavior the model abstracts away).
2. Re-check the implementation against the model: update each changed action to mirror the code as shipped, with its `file:line`, re-run TLC, and confirm the control still fails. If the code and the model disagree, the model result doesn't cover the code.

Let the fix fit the bug; the smallest change that closes the trace is usually right. Never weaken an invariant to get green unless the user agrees the business rule was wrong.

## Output

If the user wants the spec kept, commit it next to the code (e.g. `specs/<machine>/`) with the `.tla`, `.cfg` files (each model's control included), state diagram, a short README covering what's modeled, what's abstracted away, and how to run it, and a script that runs every config and prints holds or violated. Check the repo's own rules for where non-code files may go first. Gitignore TLC's `states/` directory and `*_TTrace_*` files.

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
- Fix: <proposed change; if made, whether the model of the shipped code passes>

## Checked and safe
<each candidate the model refuted, and the guard that makes it safe (with its control result)>

## Leads
<suspicions found while reading but not modeled or reproduced>

## Modeling notes
<abstractions, what is NOT covered, constants, state counts>
```

Rank findings by impact, and say what the model leaves out (other services, DB isolation level, clock skew) so a clean run isn't mistaken for proof the whole system is correct.
