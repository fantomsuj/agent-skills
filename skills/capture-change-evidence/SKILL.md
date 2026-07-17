---
name: capture-change-evidence
description: Capture and report before-and-after evidence for architectural and UI decisions made during implementation. Use when building, changing, refactoring, or reviewing features that affect system structure, APIs, data flow, permissions, dependencies, user interfaces, interaction states, responsive behavior, or other observable product behavior; when a user requests a change report, implementation record, screenshots, design rationale, architectural decision log, or proof of what changed; or when work should preserve a verifiable baseline before edits begin.
---

# Capture Change Evidence

Create an evidence trail while doing the work, not as a reconstruction afterward. Capture the baseline before the first relevant edit, record decisions as they occur, capture matching after states, and publish a concise report.

## Non-negotiable rule

Pause before editing source files and establish the baseline.

If the skill is invoked after work has started, do not mislabel the current state as “before.” Reproduce the baseline from the target base commit in a temporary worktree when practical. If it cannot be reproduced, state exactly which evidence is unavailable and why.

## Choose the evidence tracks

Use every track affected by the task:

- **Architecture:** modules, dependencies, APIs, schemas, persistence, data flow, permissions, security boundaries, background jobs, deployment, configuration, or performance behavior.
- **UI:** layout, styling, content, components, states, interactions, navigation, motion, accessibility, or responsive behavior.

Do not manufacture UI evidence for architecture-only work or architecture diagrams for a trivial visual tweak. For work affecting both tracks, complete both.

## 1. Create the evidence workspace

Choose a short slug and use:

```text
.context/change-evidence/<slug>/
  baseline.md
  decisions.md
  before/
  after/
```

Keep temporary logs and exploratory captures in `.context`. Publish curated evidence at the end under the user’s requested location or, by default:

```text
reports/change-evidence/<slug>/
  report.md
  screenshots/
```

Record in `baseline.md`:

- branch, target base, baseline commit, and working-tree status;
- relevant routes, feature flags, accounts/roles, seed data, viewport, theme, and browser;
- commands needed to run the baseline;
- affected architecture and UI surfaces;
- known baseline failures or environmental limitations.

Use the repository’s configured base branch. Determine the merge base rather than assuming the local default branch is current.

## 2. Capture before evidence

Complete this step before editing relevant source files.

### Architecture baseline

Capture the smallest evidence set that makes the old design verifiable:

- relevant file and module map;
- current API, schema, permission, configuration, or dependency surface;
- current data/control flow when three or more components interact;
- focused test or command output that characterizes existing behavior;
- baseline commit and exact commands used.

Prefer a compact table or Mermaid diagram when relationships are otherwise hard to understand. Do not paste large source files into the report.

### UI baseline

Capture each state the requested change will visibly alter. Use deterministic demo or seeded data and avoid secrets or personal information.

For every screenshot, record:

- route or entry point;
- viewport dimensions and device scale when relevant;
- theme, locale, role, feature flags, and seeded state;
- browser or rendering environment;
- capture filename.

Name captures in stable pairs:

```text
01-default-before.png
01-default-after.png
02-error-before.png
02-error-after.png
```

Use the same route, viewport, theme, data, scroll position, and interaction state for the matching after capture. Capture responsive, loading, empty, error, hover, focus, modal, or permission states only when the change affects them.

Open and inspect every screenshot after capture. Retake images that are blank, clipped, obscured, nondeterministic, or contain sensitive data.

## 3. Record decisions during implementation

Append decisions to `decisions.md` when they are made. For each material architectural or UI decision, record:

```text
Decision:
Context:
Options considered:
Choice and rationale:
Consequences and tradeoffs:
Evidence to capture after implementation:
```

Record decisions that change boundaries or user behavior. Skip routine syntax and mechanical edits.

For architecture, pay attention to contracts, ownership, failure behavior, trust boundaries, migration, compatibility, observability, and performance.

For UI, pay attention to hierarchy, content, layout, components, interaction feedback, accessibility, responsive behavior, and visual-system consistency.

## 4. Implement and validate

Make the requested change. Preserve the baseline artifacts while working.

Run checks proportional to risk, including focused tests and the project’s standard lint, typecheck, build, or integration commands. Exercise the riskiest changed path directly.

Do not treat a passing build as UI evidence or a screenshot as architectural validation.

## 5. Capture matching after evidence

Repeat the baseline commands and UI scenarios under the same conditions.

For architecture, capture:

- the resulting module, dependency, API, schema, permission, configuration, or data-flow surface;
- focused behavior and test output;
- migration or compatibility results when applicable;
- final diff statistics and changed-file map.

For UI, capture every paired state using the matching `*-after.png` name. Add a new unpaired after screenshot only for a genuinely new state that did not exist before, and label it as new.

Compare each pair. Verify that the image demonstrates the intended change and does not reveal an unrelated regression.

## 6. Publish the report

Copy `assets/change-report-template.md` into the final evidence directory and replace every placeholder. Keep the report concise and evidence-led.

Include:

- scope, baseline, branch, and comparison target;
- executive summary;
- architectural decisions with tradeoffs;
- before/after architecture evidence;
- paired UI screenshots with captions explaining the visible change;
- changed-file map;
- validation commands and results;
- limitations, missing evidence, warnings, and follow-ups.

Link to local files with paths that render from the report location. Do not claim a before/after comparison when either side is missing.

## Completion gate

Do not call the work complete until every applicable item is true:

- baseline metadata was captured before edits or honestly reconstructed from a known commit;
- every material decision has rationale and consequences;
- architecture changes have before and after evidence;
- every changed UI state has a verified screenshot pair under matching conditions;
- validation results are recorded;
- the report distinguishes committed, staged, and uncommitted work;
- sensitive or personal data is absent from artifacts;
- limitations and missing evidence are explicit.

In the final response, link the report and summarize the strongest evidence and any unresolved limitation.
