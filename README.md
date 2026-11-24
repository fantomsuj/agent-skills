# Agent skills

Skills for coding agents and chatbots. Each skill is a directory under `skills/` containing a `SKILL.md` file with instructions for the agent, plus an optional `agents/openai.yaml` for Codex-style harnesses.

## Skills

- **[capture-change-evidence](skills/capture-change-evidence/SKILL.md):** records before-and-after evidence and design decisions while making architectural or UI changes.
- **[in-app-feedback-notes](skills/in-app-feedback-notes/SKILL.md):** builds an annotate-anything feedback layer into a web app. People point at an element in the running app and leave a note; the note is committed to the repo as a markdown file with a screenshot crop, drawn back as a mark on that element, and worked by a coordinator agent that claims, fixes and archives it in PRs. Includes the file contract, capture and anchoring details, the GitHub commit path, trust rules for outside users, and a coordinator skill to drop into the target repo.
- **[tla-state-machine-bugs](skills/tla-state-machine-bugs/SKILL.md):** finds lifecycle and race-condition bugs by extracting implicit state machines from code, modeling them in TLA+, and model-checking with TLC. Draws on Tobi Lütke's "verification engineering" approach and Shopify Engineering's "Why Developers Should Be Force-Fed State Machines".

## External skills I use

- **[socratic-tutor](https://github.com/janestreet/agent_skills/tree/main/socratic-tutor)** (Jane Street, MIT): a strict Socratic tutor with a diagnostic, a skill map with levels 0 to 5, one question at a time, spaced review, and a final assessment. Triggers on "teach me", "tutor me", "quiz me". Useful for learning TLA+ itself before running `tla-state-machine-bugs`. Jane Street doesn't accept changes to it, so install it from their repo rather than copying it here.

## How to use

With a coding agent that supports the `SKILL.md` format, copy a skill directory into your agent's skills folder (e.g. `~/.claude/skills/`). With a plain chatbot, paste the body of the `SKILL.md` into the chat or a system prompt.
