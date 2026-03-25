# AGENTS.md

## Review instructions
When the user asks for a code review / PR review / diff review, use the `code-review` subagent configured in `agents/code-review.md`.

## Five Whys instructions
When the user asks for a 5 Whys analysis, use the `five-whys` subagent configured in `agents/five-whys.md`.

## Changes summary instructions
When the user asks for a changes summary, use the `changes-summary` subagent configured in `agents/changes-summary.md`.

## Pull Request instructions
When the user asks to make a pull request, use the `pull-request` subagent configured in `agents/pull-request.md`.

## Planning instructions
When the user asks to plan an implementation or requests a step-by-step plan before coding, load the `planning-style` skill configured in `skills/planning-style/SKILL.md` via `eca__skill` before writing the plan.

## Commit instructions
When the user asks to create a commit or uses the `/commit` command, use the `commit` subagent configured in `agents/commit.md`.

## Implementation instructions
When the user asks to implement a plan, write code, refactor, or apply changes, use the `implement` subagent configured in `agents/implement.md`.

## QA check instructions
After an agent produces or modifies code, use the `qa-check` subagent configured in `agents/qa-check.md` to run linting and unit tests before considering the work complete.
