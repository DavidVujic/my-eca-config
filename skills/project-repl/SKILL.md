---
name: project-repl
description: Use when a focused Python question (behavior of a function, library call, data shape, edge case) is quicker to answer by running code than by reading it, before or while editing project code. Covers when to use the REPL, the experiment flow, and telling normal exceptions apart from unexpected MCP or kernel failures.
---

# Project REPL Skill

Applies to the parent agent and every subagent. Guidance for focused Python experiments with `agent-repl-mcp`; the server's own instructions cover the `run`/`close` mechanics, persistence, and resets.

## When to use

- Prefer `agent-repl-mcp__run` over long `python -c` commands or temporary project copies for focused Python exploration.
- Skip it when reading the code already answers the question.
- Never install or modify dependencies to make the REPL work.

## Procedure

1. Import the project code under test.
2. Try the behavior with real inputs; narrow down with further small calls.
3. When useful, sketch a fix as an in-memory redefinition and compare results.
4. An in-memory experiment is not saved: make the physical source edit with the normal edit tools, then check it in the REPL. Redefine remaining sketches back or reset the kernel first so they do not shadow the edited source.
5. Clean up as described in "Session cleanup".

## Session cleanup

- Pass the same absolute `project_full_path` to every `run` call in the task; a different path starts a different kernel.
- Keep the kernel alive while experiments continue; do not close it between experiments. Close mid-task only to reset state deliberately.
- After the final experiment, call `agent-repl-mcp__close` with the same `project_full_path`, before the parent ends its turn or a subagent returns. Close it also when the last experiment failed or was abandoned.
- If `close` errors, report the error and the path in your summary; do not claim the kernel was cleaned up.

## Errors

- A Python exception raised by your experiment (`ValueError`, `ImportError` from your own code, a failing assertion) is normal output. Read it and continue.
- Unexpected MCP or kernel behavior is different: tool errors, a dead or hung kernel, lost state, wrong `project_full_path` handling, or results that still contradict the code after a kernel reset (which rules out a leftover in-memory redefinition). Report it to the developer promptly with the call made, the expected result versus the actual result or error, and no secrets. Do not silently work around it (for example by switching to `python -c` or a project copy) without saying so.

## Limits

- The REPL does not replace lasting regression tests; add or update tests in the project when behavior changes.
- It does not replace the required lint and unit tests, which `qa-check` owns.
