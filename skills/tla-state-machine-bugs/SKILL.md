---
name: tla-state-machine-bugs
description: Find state-machine bugs in a codebase by extracting the implicit state machines hidden in the code, writing them out explicitly, modeling them in TLA+, and model-checking with TLC to get concrete counterexample traces. Use this skill whenever the user asks to find bugs in lifecycle/status logic (orders, payments, subscriptions, jobs, sessions, connections, onboarding, sync), mentions race conditions, double-processing, stuck records, impossible states, retries, webhooks, idempotency, TLA+, PlusCal, TLC, formal verification or model checking, or asks to 'make the state machine explicit' — even if they never say 'TLA+'.
---

# Finding state-machine bugs with TLA+

Most production bugs in stateful code aren't typos — they're transitions nobody thought about: a cancel that lands after a charge, a retry that runs twice, a webhook that arrives before the row exists. These bugs live in *implicit* state machines: status columns, boolean flags, nullable timestamps, and the scattered code paths that write them. No single file shows the whole machine, so code review rarely catches them.

The strategy (popularized by Tobi Lütke as "verification engineering") is:

1. **Make the implicit state machine explicit.** Just writing it down surfaces many bugs.
2. **Model it in TLA+** so every reachable state, interleaving, and failure is explored mechanically.
3. **Model-check it**, turn each counterexample into a reproducing test, then fix and re-verify.

The ways to spot implicit machines and the guidance on recording transitions draw on Shopify Engineering's "Why Developers Should Be Force-Fed State Machines".

Your job is to run this loop and come back with *confirmed* bugs, each with a trace, a repro, and a location in the code — not a pile of speculation.

## Phase 0 — Scope

Pick one bounded component with a lifecycle. Good targets: anything with a `status`/`state` field, a queue consumer, a payment/refund flow, a retry loop, a sync between two systems, an auth/session flow. If the user pointed at a whole repo, list candidate machines (search for status enums, `state =`, `is_*`/`has_*` flags such as `is_published` or `paid`, nullable `*_at` timestamps such as `published_at`, start/end date pairs on records that are only valid for a period (subscriptions, trials, holds), `transition`, `retry`, `webhook`, and `delete`/`destroy` calls on lifecycle entities) and start with the one where concurrency or money is involved. Tell the user which one you picked and why.

Model one machine at a time. A spec that tries to cover the whole system never finishes checking and nobody can read it.

Match the effort to the machine. A record with two states and one writer only needs the Phase 1 table, so skip TLA+ there. TLA+ pays off when there are several writers, steps that aren't atomic, external systems, or retries.

## Phase 1 — Extract the explicit state machine

Read the code and build this inventory before writing any TLA+. It's the most valuable artifact in the whole process, and the spec will only be as good as it is.

**State variables.** Every field that encodes where an entity is in its life: enums, booleans, nullable timestamps, counters, foreign keys whose presence matters, plus *external* state (the payment provider's view, a cache, a message queue). The real state is the product of all of these, so note combinations the code implicitly assumes can't happen (e.g. `shipped_at` set while `status = "pending"`).

Two kinds of state are easy to miss:

- **Time.** A record with start and end dates changes state when the clock passes the end date, even though no code path writes anything. Find out what actually acts on expiry: a cron job, a check when the record is read, or nothing. Model expiry as its own action (`Expire(o)`) that can fire at any point, including in the middle of other transitions.
- **Deletion.** A hard delete is a transition into an implicit "gone" state that nothing can leave. It erases the history and can leave behind dangling references, queued jobs, and webhooks for an entity that no longer exists. Model `Delete` as a transition and check what can still act on the entity afterward.

**Transitions.** Every code path that writes a state variable. For each, record:

| Transition | From (guard) | Writes | Trigger | Atomic? | Location |
|---|---|---|---|---|---|
| Pay | status = pending | status:=paid, charged:=true | POST /pay | no: charge, then DB write | `orders/service.py:142` |

- **Guard**: what the code actually checks, not what it should check. A missing check is a finding.
- **Trigger**: user action, webhook, cron, queue message, timeout, admin tool, migration.
- **Atomic?**: Is the read-check-write in one transaction or lock? If not, the transition is really several steps that can interleave with others. This is where most race bugs come from.

**Environment.** What the outside world can do: deliver messages twice, out of order, or never; crash between two writes; run two workers at once; retry after a timeout that actually succeeded.

Then write the machine out as a list of states and allowed transitions (a Mermaid `stateDiagram-v2` works well) and look for findings before going further:

- States with no way out (stuck records)
- States the code checks for but nothing can produce (dead code, or a missing transition)
- Transitions whose guard is missing or weaker than the others into the same state
- Flag combinations that are reachable but that no code handles
- Two writers to the same field with no coordination
- Time-based states that nothing enforces (an expired subscription that still grants access)
- Hard deletes of entities that other records, jobs, or webhooks can still reach
- Internal states leaking into an external API, or API clients able to request transitions the machine shouldn't allow

**Check history if it exists.** Some systems already record transitions: an audit log, a transitions table, event-sourcing streams, or per-state timestamps. If yours does, it gives you evidence from production. Ask the user before querying anything outside a dev environment. Then look for logged transitions that aren't in your table, entities that reached a state with no legal path to it, and entities that have sat in a non-terminal state for an unusually long time. A hit here is stronger evidence than any model, because it already happened.

Record each as a **candidate**. Phase 2 confirms or rejects them.

## Phase 2 — Write the TLA+ spec

The spec should mirror the **code**, not the design doc or what the code was meant to do. If you model the intended behavior, TLC will verify the intent and miss every bug. When code and intent disagree, model the code and write the intended behavior down as an invariant.

Structure:

- `VARIABLES`: one per state variable from Phase 1, usually functions over a small set of entities (`status \in [Orders -> States]`).
- One action per transition. Name it after the code path and put the `file:line` in a comment, so a trace can be mapped back to the code.
- **Split non-atomic transitions** into separate actions with an intermediate program-counter value (`pc[o] = "charged_not_saved"`). This is what lets TLC find interleavings.
- Model the environment as actions too: `Crash`, `DuplicateDeliver`, `Reorder`, `Timeout`. These are where the real bugs hide.
- **Invariants** (safety, "nothing bad ever happens"): `TypeOK`, plus properties stated in business terms, such as never charged twice, never shipped while unpaid, refund at most what was charged.
- **Temporal properties** (liveness, "something good eventually happens") only where they matter, such as every paid order eventually ships or refunds. These need fairness (`WF_vars(Action)`) on the actions the system guarantees will run.

Keep constants tiny: 2–3 entities, retries ≤ 2. Most concurrency bugs show up with 2 actors, and the state space grows exponentially.

PlusCal is fine if the logic reads more naturally as sequential code with labels. Each label is one atomic step, so put labels exactly where the real code can be interleaved.

### Skeleton

```tla
---- MODULE OrderLifecycle ----
EXTENDS Naturals, TLC
CONSTANTS Orders
VARIABLES status, charges
vars == <<status, charges>>

States == {"pending", "paid", "shipped", "cancelled", "refunded"}

TypeOK == /\ status  \in [Orders -> States]
          /\ charges \in [Orders -> 0..3]

Init == /\ status  = [o \in Orders |-> "pending"]
        /\ charges = [o \in Orders |-> 0]

\* orders/service.py:142  pay() -- charge API call, then DB write (not atomic in code)
Pay(o) == /\ status[o] = "pending"
          /\ charges' = [charges EXCEPT ![o] = @ + 1]
          /\ status'  = [status  EXCEPT ![o] = "paid"]

\* orders/service.py:201  cancel() -- note: code does NOT check charges
Cancel(o) == /\ status[o] \in {"pending", "paid"}
             /\ status' = [status EXCEPT ![o] = "cancelled"]
             /\ UNCHANGED charges

Next == \E o \in Orders : Pay(o) \/ Cancel(o)
Spec == Init /\ [][Next]_vars /\ WF_vars(Next)

\* Business rule: a cancelled order is never left holding the customer's money
NoChargeOnCancelled == \A o \in Orders : status[o] = "cancelled" => charges[o] = 0
====
```

```
\* OrderLifecycle.cfg
CONSTANTS Orders = {o1, o2}
SPECIFICATION Spec
INVARIANTS TypeOK NoChargeOnCancelled
```

## Phase 3 — Model-check

Get the tools if they're missing: `tla2tools.jar` from the tlaplus/tlaplus GitHub releases (needs Java 11+). Run:

```bash
java -XX:+UseParallelGC -cp tla2tools.jar tlc2.TLC -workers auto -deadlock -config Spec.cfg Spec.tla
```

(`-deadlock` turns off deadlock reporting. Leave it out when you *want* stuck states reported, which you usually do for lifecycles that should always progress. Then add explicit "terminal" self-loops for states that are legitimately final.)

If TLC runs too long, shrink constants before anything else, then add `SYMMETRY` for interchangeable entities. Apalache is an alternative for specs with large integer ranges, but TLC is the default.

**Guard against a spec that proves nothing.** A passing check only means something if the invariant could have failed. Before trusting a green run:

- Temporarily add a false invariant (e.g. `status[o1] /= "shipped"`) and confirm TLC reaches it. If not, your actions are too restricted and states you care about are unreachable.
- Check the distinct-states count TLC reports. A handful of states for a non-trivial machine means you over-constrained `Init` or the guards.

**Read each counterexample** as a story: "order o1 is pending → user clicks pay → charge succeeds → user clicks cancel before the DB write → ...". Map every step back to its `file:line` comment.

## Phase 4 — Confirm, then fix

A TLC trace shows a bug **in the model**. Before calling it a bug in the code:

1. Check that each step in the trace corresponds to something the real code can do. If it doesn't, your model is too loose: tighten it and note it as a modeling correction, not a finding.
2. Write a failing test that reproduces the trace: a unit test that interleaves calls, an integration test with a duplicated webhook, or a test that forces a crash between two writes.
3. Only then report it as **confirmed**. Anything you couldn't reproduce stays **suspected**, with the reason.

When fixing, change the spec first (add the guard, make the step atomic, add an idempotency key), re-run TLC until it's green *and* the false-invariant check still trips, then make the matching code change and confirm the repro test passes. Don't make the invariant weaker to get a green run unless the user agrees the business rule was wrong.

Fixes that make the machine more explicit tend to stay fixed:

- **Replace flag combinations with one explicit state** where the code is effectively treating several booleans as a status field.
- **Model removal as a state instead of a delete.** Mark records `spam`, `fraud`, or `cancelled_by_admin` rather than deleting them. This keeps them accountable and keeps their history.
- **Record transitions.** If the machine never revisits a state, a timestamp per state is enough (`paid_at`, `shipped_at`). If states can recur, use a log table with the timestamp, old state, new state, and the event that triggered it, because per-state fields get overwritten. A log also lets you validate the spec against real behavior later, and answer questions like how long fraud takes to detect.
- **Use the stack's state-machine library** where one exists (e.g. Ruby's `state_machines` gem or XState), so transitions are declared in one place and illegal ones fail loudly. Keep the TLA+ spec in step with the declared machine.

## Output

Commit the spec next to the code (e.g. `specs/<machine>/`) with the `.tla`, the `.cfg`, the state diagram, and a short README that says what's modeled, what's abstracted away, and how to run it. A spec that lives in the repo gets updated when the code changes. A spec in a chat log goes stale.

Report to the user in this shape:

```markdown
# State-machine review: <component>

## The machine
<state diagram + transition table, condensed>

## Findings
### 1. <one-line title>  — confirmed | suspected
- Trace: <numbered steps, plain English>
- Code: <file:line for each step>
- Impact: <what goes wrong for users/money/data>
- Repro: <test name or why it couldn't be reproduced>
- Fix: <proposed change, and whether the spec now passes with it>

## Modeling notes
<abstractions made, what is NOT covered, constants used, state counts>
```

Rank findings by impact. Be explicit about what the model leaves out (other services, the DB isolation level, clock skew), so a clean run isn't mistaken for proof that the whole system is correct.

## Optional later steps

The full strategy continues with a security review of the component and a rewrite in a memory-safe language, checked against the spec and acceptance tests. Only do these if the user asks. If they do, the spec becomes the acceptance criterion: the rewrite has to keep every invariant green, and the repro tests from Phase 4 become part of its test suite.
