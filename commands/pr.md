# pr

Create a pull request for the current feature branch.

This command is an explicit request for a pull request. Follow "Pull Request instructions" in `AGENTS.md`: the parent loads the `creating-pull-request` skill via `eca__skill` and runs it; it runs the Git and Code Health preflight, the `pull-request` subagent only drafts the title and body, and the parent pushes and creates the PR.

Arguments: $ARGUMENTS
