# Change evidence report: <title>

**Branch:** `<branch>`

**Compared with:** `<target base and baseline commit>`

**Prepared:** `<date>`

**Evidence tracks:** `<Architecture | UI | Architecture and UI>`

## Executive summary

<What changed, why it changed, and the user/developer impact.>

## Scope and capture conditions

| Item | Value |
|---|---|
| Baseline source | `<commit or live environment>` |
| Working-tree state | `<clean, staged, or uncommitted details>` |
| App entry point | `<route, page, command, or N/A>` |
| UI conditions | `<viewport, theme, role, seed data, browser, or N/A>` |

## Decisions

### <Decision title>

- **Context:** <why a decision was needed>
- **Options considered:** <meaningful alternatives>
- **Decision:** <chosen approach and rationale>
- **Consequences:** <tradeoffs, compatibility, risks, or follow-ups>

## Architecture evidence

| Surface | Before | After | Evidence |
|---|---|---|---|
| `<module, API, schema, permission, data flow, etc.>` | `<baseline behavior>` | `<resulting behavior>` | `<file, diagram, test, or command>` |

<Add a small Mermaid diagram only when it makes a multi-component relationship easier to verify. Remove this section for UI-only work.>

## UI evidence

### <State name>

| Before | After |
|---|---|
| ![Before: <state>](screenshots/<state>-before.png) | ![After: <state>](screenshots/<state>-after.png) |

<Explain the visible change and confirm that route, viewport, data, theme, and interaction state match. Repeat for each affected state. Remove this section for architecture-only work.>

## Changed-file map

| Area | Files | Purpose |
|---|---|---|
| `<area>` | `<paths>` | `<reason>` |

## Validation

| Check | Result |
|---|---|
| `<command or manual scenario>` | `<pass/fail and key output>` |

## Limitations and follow-ups

- <Missing evidence, environment differences, warnings, remaining risk, or “None.”>
