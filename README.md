# my-eca-config

Shareable ECA (Editor Code Assistant) configuration focused on agent orchestration, subagents, and reusable skills.

Contents
- `AGENTS.md` — central, human-readable definitions for primary agents, their responsibilities, and global guidance for use.
- `agents/` — subagent configurations (implement, code-review, changes-summary, pull-request, five-whys, explorer, general, etc.) with task-specific instructions and expected outputs.
- `skills/` (or inline skill docs) — small, repeatable procedures that agents invoke to perform specialized work.

Purpose
- Provide a compact, explicit blueprint for running agent-driven workflows and for reusing those patterns in other automation systems.
- Files are plain Markdown and small config fragments so the agent designs and skill interfaces are easy to read, adapt, and port to other tools or CI pipelines.
