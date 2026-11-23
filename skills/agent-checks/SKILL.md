---
name: agent-checks
description: Use before any `agent-python-checks-mcp` call, for lint, typecheck and test during implementation or on a direct check request. Covers the stable `project_full_path` and `session_id`, watcher startup, reuse and cleanup, polling `running`, reading `origin`, unavailable checks, and reporting.
---

# Agent Checks Skill

Applies to the parent agent and every subagent. Single source for how to use `agent-python-checks-mcp` (`start_watching`, `lint`, `typecheck`, `test`, `stop_watching`); the server's own instructions cover the tool signatures. Host-specific routing and how the parent obtains a `session_id` stay in `AGENTS.md`.

## Session values

- `project_full_path`: the known absolute root of the target project, unchanged for every call in the task. If you do not know it, ask the user (parent) or report it missing (subagent); never guess or fabricate a path.
- `session_id`: one stable id per checks session, passed unchanged to every checks call in the same task. If the parent supplied it, use it as given; never generate, alter, or guess it as a subagent.
- Checks sessions active at the same time need distinct ids. A later sequential checks session may reuse an id only after the previous one has stopped its watcher.
- If either value is missing, do not call the checks tools; report that the checks session was not started and continue the task.

## Implementation session (watcher)

1. Start: call `start_watching` with both values before the first edit. It returns at once with the detected commands; the initial runs continue in the background.
2. Resume: if your earlier `start_watching` with the same values is still active, reuse it. If you already called `stop_watching`, or the session was lost, start it again with the same values.
3. Failure: if `start_watching` fails, report the error and continue the task; still attempt the final checks, which run on demand.
4. Final checks: after your last edit, call `lint`, `typecheck` and `test` with both values (see "Reading results").
5. Cleanup: call `stop_watching` with the same values after the final results, even when checks fail. On any early exit (rejected edit, blocker) once a session was started, stop it as a best effort.

## Direct check requests (no watcher)

- When the user asks to check code, call the requested check tools on demand: only the named check(s), or all three (`lint`, `typecheck`, `test`) for a general check request.
- Do not call `start_watching` or `stop_watching`, and do not spawn a subagent for it.
- Read and report the results as below.

## Reading results

- `running: true` is never a current pass, even when it carries an older cached result. Call the same check again until `running: false`; if it cannot finish, report the check unavailable.
- Count a pass only for a successful result produced after your last change. A cached result from before your edits is not current.
- `origin` tells where the result came from: `watcher` (rerun after file changes), `initial` (the run started by `start_watching`), `on-demand` (run by this call), or `none` (no command of that kind was detected).
- Unavailable, never passed: a check that errors, returns `origin: "none"`, has no detected command, or a failed or cancelled run without results.
- Do not run lint, typecheck or test through the shell, and do not guess or substitute commands for an unavailable check.
- Failing checks do not block the report; never claim success for a check that did not pass.

## Reporting

- Startup: started, reused, not started (missing values), or failed (with the error). Skip for direct check requests.
- Per check: `lint`, `typecheck`, `test` as pass, fail (with failures), or unavailable (with the reason), each with its `origin`.
- Cleanup: stopped, or the `stop_watching` error. Skip for direct check requests.
