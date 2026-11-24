# TLA+ reference for state-machine specs

Read this when you reach Phase 2 (writing the spec) or Phase 3 (running TLC).

## Spec structure

- `VARIABLES`: one per state variable from the Phase 1 inventory, usually functions over a small set of entities (`status \in [Orders -> States]`).
- One action per transition, named after the code path, with its `file:line` in a comment so traces map back to code.
- Non-atomic transitions split into separate actions with an intermediate program-counter value (`pc[o] = "charged_not_saved"`). This is what lets TLC find interleavings.
- Environment actions: `Crash`, `DuplicateDeliver`, `Reorder`, `Timeout`, `Expire`, `Delete`.
- Invariants (safety): `TypeOK` plus business rules, such as never charged twice, never shipped while unpaid, refund at most what was charged.
- Temporal properties (liveness) only where they matter, such as every paid order eventually ships or refunds. These need fairness (`WF_vars(Action)`) on actions the system guarantees will run.
- Constants kept tiny: 2 or 3 entities, retries at most 2. Most concurrency bugs show up with 2 actors.

PlusCal is fine when the logic reads more naturally as sequential code. Each label is one atomic step, so put labels exactly where the real code can be interleaved.

## Skeleton

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

## Running TLC

Get `tla2tools.jar` from the tlaplus/tlaplus GitHub releases (needs Java 11+), then:

```bash
java -XX:+UseParallelGC -cp tla2tools.jar tlc2.TLC -workers auto -config Spec.cfg Spec.tla
```

TLC reports deadlocks by default, which is usually what you want for lifecycles that should always progress. Add explicit self-loops for states that are legitimately final, or pass `-deadlock` to turn the check off.

If a run takes too long, shrink constants first, then add `SYMMETRY` for interchangeable entities. Apalache is an alternative for specs with large integer ranges.
