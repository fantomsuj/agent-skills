# TLA+ reference for state-machine specs

Read this when you reach Phase 2 (writing the spec) or Phase 3 (running TLC).

## Spec structure

- `VARIABLES`: one per state variable from the Phase 1 inventory, usually functions over a small set of entities (`status \in [Jobs -> States]`).
- One action per transition, named after the code path, with its `file:line` in a comment so traces map back to code.
- Non-atomic transitions split into separate actions with an intermediate program-counter value (`pc[j] = "claimed_not_saved"`). This is what lets TLC find interleavings.
- Environment actions: `Crash`, `DuplicateDeliver`, `Reorder`, `Timeout`, `Expire`, `Delete`.
- Invariants (safety): `TypeOK` plus the rules the domain depends on, such as at most one worker runs a job, nothing is published before approval, a revoked token is never accepted, a deleted record is never written to.
- Temporal properties (liveness) only where they matter, such as every queued job eventually finishes. These need fairness (`WF_vars(Action)`) on actions the system guarantees will run.
- Constants kept tiny: 2 or 3 entities, retries at most 2. Most concurrency bugs show up with 2 actors.

PlusCal is fine when the logic reads more naturally as sequential code. Each label is one atomic step, so put labels exactly where the real code can be interleaved.

## Skeleton

A background job queue where a watchdog re-queues slow jobs without stopping the worker that holds them. TLC finds the trace: w1 claims j1, the watchdog re-queues it, w2 claims it while w1 is still running.

```tla
---- MODULE JobQueue ----
EXTENDS FiniteSets, TLC
CONSTANTS Jobs, Workers
VARIABLES status, runners
vars == <<status, runners>>

States == {"queued", "running", "done"}

TypeOK == /\ status  \in [Jobs -> States]
          /\ runners \in [Jobs -> SUBSET Workers]

Init == /\ status  = [j \in Jobs |-> "queued"]
        /\ runners = [j \in Jobs |-> {}]

\* jobs/worker.py:88  claim()
Claim(w, j) == /\ status[j] = "queued"
               /\ status'  = [status  EXCEPT ![j] = "running"]
               /\ runners' = [runners EXCEPT ![j] = @ \cup {w}]

\* jobs/watchdog.py:31  requeue_stale() -- note: does NOT stop the running worker
Timeout(j) == /\ status[j] = "running"
              /\ status' = [status EXCEPT ![j] = "queued"]
              /\ UNCHANGED runners

\* jobs/worker.py:120  finish()
Finish(w, j) == /\ w \in runners[j]
                /\ status'  = [status  EXCEPT ![j] = "done"]
                /\ runners' = [runners EXCEPT ![j] = @ \ {w}]

\* Legitimate final state: stutter so TLC doesn't report it as a deadlock
Terminated == /\ \A j \in Jobs : status[j] = "done"
              /\ UNCHANGED vars

Next == \/ \E j \in Jobs : Timeout(j) \/ \E w \in Workers : Claim(w, j) \/ Finish(w, j)
        \/ Terminated
Spec == Init /\ [][Next]_vars /\ WF_vars(Next)

\* Domain rule: a job is never executed by two workers at once
AtMostOneRunner == \A j \in Jobs : Cardinality(runners[j]) <= 1
====
```

```
\* JobQueue.cfg
CONSTANTS Jobs = {j1}
          Workers = {w1, w2}
SPECIFICATION Spec
INVARIANTS TypeOK AtMostOneRunner
```

## Running TLC

Get `tla2tools.jar` from the tlaplus/tlaplus GitHub releases (needs Java 11+), then:

```bash
java -XX:+UseParallelGC -cp tla2tools.jar tlc2.TLC -workers auto -config Spec.cfg Spec.tla
```

TLC reports deadlocks by default, which is usually what you want for lifecycles that should always progress. Add a stuttering action for states that are legitimately final (like `Terminated` above), or pass `-deadlock` to turn the check off.

If a run takes too long, shrink constants first, then add `SYMMETRY` for interchangeable entities. Apalache is an alternative for specs with large integer ranges.
