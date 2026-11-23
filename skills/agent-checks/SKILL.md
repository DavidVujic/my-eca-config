---
name: agent-checks
description: Use before any `agent-python-checks-mcp` call, for lint, typecheck and test during implementation or on a direct check request. Covers the stable `project_full_path` and `session_id`, watcher startup, reuse and cleanup, polling `running`, reading `origin`, unavailable checks, and reporting.
---

# Agent Checks Skill

How to run lint, typecheck and test through `agent-python-checks-mcp`.

## Session values

- `project_full_path`: the known absolute project root; `session_id`: one stable id per checks session. Pass both unchanged to every checks call; never guess, alter, or fabricate either.
- Concurrent sessions need distinct ids; a sequential session may reuse an id only after the previous watcher stopped.
- If either value is missing, call no checks tools; report the session as not started and continue.

## Implementation session (watcher)

1. Call `start_watching` before the first edit. Reuse an active watcher with the same values; restart a stopped or lost one. If startup fails, report the error and continue; the final checks still run on demand.
2. After the last edit, call `lint`, `typecheck` and `test`.
3. Always call `stop_watching` after the final results, even when checks fail; on early exit (rejected edit, blocker), stop it as a best effort.

## Direct check requests

Call only the requested checks, or all three for a general request, on demand without `start_watching` or `stop_watching`.

## Reading results

- `running: true` is not a current pass, even with an older cached result; call again until `running: false`, or report the check unavailable.
- Pass only for a successful result produced after your last change.
- Report completed failures as fail. Errors, `origin: "none"`, no detected command, or a failed or cancelled run without results count as unavailable, never passed.
- Never fall back to shell commands or substitute commands.

## Reporting

- Startup (watcher only): started, reused, not started, or failed with the error.
- Per check: pass, fail with failures, or unavailable with the reason, each with its `origin`.
- Cleanup (watcher only): stopped, or the `stop_watching` error.
