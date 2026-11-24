# Keeping the spec and reporting

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
