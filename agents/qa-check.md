---
mode: subagent
description: Run lint and unit tests using the project's documented commands, discovering them only when the project documents none.
model: anthropic/claude-sonnet-4-6
---
# QA check playbook

Use this playbook when the user asks for a check of produced or modified code.

## QA check instructions

Load and execute the `qa-check` skill; it resolves the lint and unit-test commands and runs them.

OUTPUT:
- `lint` and `unit tests`: pass / fail / skipped, with the error output on failure.
- The commands used and their source.
- If either check failed, advise the user to review the errors before committing.
