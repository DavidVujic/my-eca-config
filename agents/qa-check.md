---
mode: subagent
description: Run lint and unit tests using the project's documented commands, asking the user for any missing command instead of guessing.
model: anthropic/claude-sonnet-5-5
---
# QA check playbook

Use this playbook when the user asks for a check of produced or modified code.

## QA check instructions

Load and execute the `qa-check` skill; it resolves the lint and unit-test commands (project-documented, else asked from the user) and runs them. Do not discover or guess commands. For a missing command, ask via `eca__ask_user` if available; otherwise stop and return `missing_commands` (lint and/or unit tests) to the parent agent, which asks the user and resumes you with the answers. Never treat a missing command as skipped before the user confirms "not applicable".

Do not probe runtime services or local infrastructure as test preflight; run the documented lint and unit-test commands directly.

The QA subagent handles lint and unit tests; leave any optional Git status, diff, or whitespace checks to the parent agent.

OUTPUT:
- `lint` and `unit tests`: pass / fail / skipped / missing command, with the error output on failure.
- The commands used and their source.
- If either check failed, advise the user to review the errors before committing.
