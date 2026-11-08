---
name: project-repl
description: Use for REPL-driven Python work (behavior of a function, library call, data shape, edge case, or a change to it) when running code in a live process answers faster than reading, before or while editing project code. Covers when to use the REPL, the edit-and-verify cycle, and telling normal exceptions apart from unexpected MCP or kernel failures.
---

# Project REPL Skill

Applies to the parent agent and every subagent. Guidance for REPL-driven Python development with `agent-repl-mcp`; the server's own instructions cover the `run`/`close` mechanics, persistence, and resets.

## When to use

- Prefer `agent-repl-mcp__run` over long `python -c` commands or temporary project copies for Python behavior work.
- Skip it when reading the code already answers the question, and for documentation, config, or trivial changes.
- Never install or modify dependencies to make the REPL work.

## Procedure

Work in one persistent process and repeat steps 3-7 per change, without stopping or restarting the kernel between cycles unless step 7 requires a reset.

1. Set up fakes first, as described in "No real I/O".
2. Import the project code under test.
3. Reproduce the behavior with representative inputs; narrow down with further small calls.
4. Try a small change as an in-memory redefinition and evaluate it against the same inputs.
5. An in-memory change is not saved: persist a successful one with the normal source-edit tools.
6. Reload or re-import the edited module as appropriate. `importlib.reload` does not always update existing references (names bound via `from x import y`, instances, closures), so re-fetch the objects you call.
7. Verify the saved implementation in the same live process, with the same fakes; set them up again after a reload or reset. Stale references or leftover in-memory sketches can shadow the saved code; if you cannot rule that out, reset the kernel (`close`, then `run`) and verify the implementation on disk.
8. Clean up as described in "Session cleanup".

## No real I/O

- Never let the code under test call a real database, file, network, subprocess, queue, or service. Loading Python modules is fine.
- Use the project's existing fakes or mocks. If there are none, build a simple in-memory fake yourself, such as a dictionary or `unittest.mock`.
- Do not install dependencies or add test infrastructure for the REPL.
- Ask the user if a fake would be too complex, or if importing the code causes real I/O that a fake set up before the import cannot prevent.

## Session cleanup

- Pass the same absolute `project_full_path` to every `run` call in the task; a different path starts a different kernel.
- Keep the kernel alive while work continues; do not close it between experiments or edit-and-verify cycles. Close mid-task only to reset state deliberately.
- After the final experiment, call `agent-repl-mcp__close` with the same `project_full_path`, before the parent ends its turn or a subagent returns. Close it also when the last experiment failed or was abandoned.
- If `close` errors, report the error and the path in your summary; do not claim the kernel was cleaned up.

## Errors

- A Python exception raised by your experiment (`ValueError`, `ImportError` from your own code, a failing assertion) is normal output. Read it and continue.
- Unexpected MCP or kernel behavior is different: tool errors, a dead or hung kernel, lost state, wrong `project_full_path` handling, or results that still contradict the code after a kernel reset (which rules out a leftover in-memory redefinition). Report it to the developer promptly with the call made, the expected result versus the actual result or error, and no secrets. Do not silently work around it (for example by switching to `python -c` or a project copy) without saying so.

## Limits

- The REPL does not replace lasting regression tests; add or update tests in the project when behavior changes.
- It does not replace the required lint and unit tests, which `qa-check` owns.
