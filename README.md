# My Eca config

Shareable Eca (Editor Code Assistant) configuration for agent-driven workflows: orchestration rules, subagents, commands, and reusable skills.

## Contents
- `AGENTS.md`: global rules for the parent agent and every subagent, and which skill or subagent owns each workflow.
- `agents/`: subagent definitions with task-specific instructions and expected outputs.
- `commands/`: slash commands (`/review`, `/commit`, `/pr`, `/qa`) that start a workflow.
- `skills/`: repeatable procedures loaded with `eca__skill`.
- `config.json`: Eca settings, including models, MCP servers, tool approvals, and built-in subagent allow-lists.

## Flows
Each flow reads: command → `AGENTS.md` section → skills the parent loads → subagent (skills the subagent loads itself).

- `/review` → "Review instructions" → `git-changes-context` (worktree) → `code-review` (loads `editor-navigation`, `structural-checks`, `fp-idiomatic-style`, `coding-style`).
- `/commit` → "Commit instructions" → `commit` skill + `safeguarding-ai-generated-code` → `commit` subagent (drafts the message only).
- `/pr` → "Pull Request instructions" → `creating-pull-request` + `git-changes-context` (branch) + `safeguarding-ai-generated-code` → `pull-request` subagent (drafts title and body only).
- `/qa` → "QA check instructions" → `qa-check` subagent (loads the `qa-check` skill).
- Implementation requests → "Implementation instructions" → `implement` (loads `fp-idiomatic-style`, `coding-style`, `editor-navigation`, `structural-checks`, `safeguarding-ai-generated-code`).

## Naming
The same name can be a command, a skill, and a subagent (`commit`, `qa-check`). "Load X via `eca__skill`" always means the skill; "spawn X" always means the subagent.
