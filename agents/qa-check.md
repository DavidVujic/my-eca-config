---
mode: subagent
description: Run lint and unit tests using the project's documented commands, asking the user for any missing command instead of guessing.
model: anthropic/claude-sonnet-5-5
---
# QA check playbook

Use this playbook when the user asks for a check of produced or modified code.

## QA check instructions

Load and execute the `qa-check` skill; it resolves the lint and unit-test commands (project-documented, else asked from the user) and runs them. Do not discover or guess commands. For a missing command, `eca__ask_user` is not available to subagents: stop and return `missing_commands` (lint and/or unit tests) to the parent agent, which asks the user and resumes you with the answers. Never treat a missing command as skipped before the user confirms "not applicable".

Runtime preflight, Git status and whitespace checks are out of scope (see `AGENTS.md`).

OUTPUT:
- `lint` and `unit tests`: pass / fail / skipped / missing command, with the error output on failure.
- The commands used and their source.
- If either check failed, advise the user to review the errors before committing.
