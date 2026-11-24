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

## The loop

1. **Extract the machine** (Phase 1). Inventory the state variables, every transition with its guard and whether it is atomic, and the environment; draw it and list candidate bugs. Writing it down often settles the question, so stop here if it does. Details: `references/extraction.md`.
2. **Model it** (Phase 2). Mirror the code, not the design doc: split each non-atomic step into its own action, model the environment as actions, and tag every action with its `file:line`. Skeleton and how to run TLC: `references/tla-reference.md`. Modeling guidance: `references/model-fidelity.md`.
3. **Model-check** (Phase 3). A green run only means something after the fidelity table, a control config that must fail, and reachability checks. Expect the first counterexamples to be modeling errors. See `references/model-fidelity.md`.
4. **Confirm** (Phase 4). A TLC trace is a bug in the model until a failing test reproduces it in the real code. Unreproduced traces stay suspected.
5. **Report.** Findings, checked-and-safe candidates, leads, and modeling notes, in the shape in `references/report.md`.

## Ask before fixing

This skill finds and reports. Changing production code, committing specs, or opening PRs is a separate step: ask the user before doing it, unless they already asked for fixes. When they do:

- Trying a fix in the spec first is only an experiment; a green model of a hypothetical fix says nothing about the code you then write.
- After changing the code, the repro test and neighboring suites must pass, and the model must be updated to mirror the shipped code, re-run, and its control must still fail.
- Let the fix fit the bug. Never weaken an invariant to get green unless the user agrees the business rule was wrong.

Full checklist: `references/model-fidelity.md`.
