# Writing a faithful model and trusting its result

Read this before writing the spec (Phase 2) and again before trusting a TLC run (Phase 3) or calling a trace a bug (Phase 4). `tla-reference.md` has the spec skeleton and how to run TLC.

## Writing the spec

Mirror the **code**, not the design doc. If you model intended behavior, TLC verifies the intent and misses every bug. Where code and intent disagree, model the code and write the intent as an invariant. Split every non-atomic transition into separate actions, model the environment (including expiry and deletion) as actions, and comment each action with its `file:line`.

For data-loss questions, abstract content to a set of edits and merges to set union, then add a history variable of everything the server ever held. The invariant "nothing the server held is lost from it" is one line and finds lost-update bugs in a handful of steps.

## Before trusting a run

**A green run is only as good as the model.** No single check proves the model is faithful, so do all of these before trusting one:

- **Fidelity table.** For each action, name the code path it mirrors (`file:line`) and what it simplifies. Then list what the model leaves out (other writers, failure modes, the isolation level, bounds). Read the list as the limits of the result, and put it in the report.
- **Control config.** A second `.cfg` that turns off the guard you believe makes it safe (the lock, the version check, the fix) must fail. That shows the model is sensitive to that guard; it does not show the model is complete.
- **Reachability.** Confirm the behaviors that matter actually occur in the model, e.g. a temporary invariant that says "a conflict never happens" or "two writers never overlap" should fail. A state that is never reached is a sign of an over-constrained `Init` or guard, and so is a suspiciously small distinct-states count.

**Expect the first counterexamples to be modeling errors.** An invariant is often stronger than what the system needs (it ignores a cleanup step, or a state a reader never looks at). Restate it as what the consumer actually relies on, and say so in the modeling notes. That is a correction, not a weakening; weakening is dropping a rule the business depends on.

**Read each counterexample** as a story ("j1 queued, w1 claims it, the watchdog re-queues it, w2 claims it while w1 is still running...") and map every step to its `file:line`.

## Confirming a trace, and checking a fix

A TLC trace is a bug **in the model**. Before calling it a bug in the code:

1. Check each step matches something the real code can do. If not, tighten the model and log it as a modeling correction.
2. Write a failing test that reproduces the trace (interleaved calls, a duplicated webhook, a forced crash between writes). The fastest route is usually the existing test suite with the network or database mock scripted to return the trace's outcomes in order (conflict, then offline, then success).
3. Only then mark it **confirmed**. Anything unreproduced stays **suspected**, with the reason.

If the user wants a fix: trying it in the spec first is a cheap experiment to see whether an idea closes the trace, nothing more. A green model of a hypothetical fix says nothing about the code you then write. After changing the code:

1. The repro test passes, and the neighboring test suites still pass (a fix that satisfies the model can break behavior the model abstracts away).
2. Re-check the implementation against the model: update each changed action to mirror the code as shipped, with its `file:line`, re-run TLC, and confirm the control still fails. If the code and the model disagree, the model result doesn't cover the code.

Let the fix fit the bug; the smallest change that closes the trace is usually right. Never weaken an invariant to get green unless the user agrees the business rule was wrong.
