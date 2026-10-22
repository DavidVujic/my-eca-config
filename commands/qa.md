# qa

Run linting and unit tests against the current repository.

Delegate to the `qa-check` subagent configured in `agents/qa-check.md`. If it returns missing commands, follow "QA check instructions" in `AGENTS.md`: ask the user, then resume or respawn `qa-check` with the answers.

Arguments: $ARGUMENTS
