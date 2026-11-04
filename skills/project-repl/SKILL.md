---
name: project-repl
description: Use when a focused Python question (behavior of a function, library call, data shape, edge case) is quicker to answer by running code than by reading it, before or while editing project code. Covers the agent-repl-mcp run/close procedure, what persists between calls, and reporting unexpected MCP or kernel behavior.
---

# Project REPL Skill

Applies to the parent agent and every subagent. REPL-driven development for focused Python exploration in the project's own virtualenv.

## When to use

- Prefer `agent-repl-mcp__run` over long `python -c` commands or temporary project copies for focused Python exploration.
- Skip it when reading the code already answers the question.
- The virtualenv is available and `ipykernel` is assumed to be installed as a dev dependency. Never install or modify dependencies to make the REPL work.

## Calls

- `agent-repl-mcp__run(data, project_full_path)`: `data` is the Python code; `project_full_path` is the absolute project root.
- Use the exact same absolute `project_full_path` on every call, and on the final `close`. A different path starts a different kernel.
- The last expression is returned; `print` output arrives as stdout.
- Send small chunks that are easy for the user to approve: one idea per call, not a long script.

## Persistence

- The Jupyter kernel persists. Imports, definitions and values stay available between calls, so import once and reuse.
- In-memory changes stick: any redefinition or patch made in the REPL (function, class, module attribute, value) affects every later `run` call and check in the same kernel, until you change it back or close the kernel.
- In-memory code is not the physical module. A redefinition or patch never changes a source file.
- To reset in-memory state, call `agent-repl-mcp__close` with the same `project_full_path`; the next `run` creates a fresh session. Alternatively, redefine the code back to its original state.
- Closing or resetting the kernel does not undo physical source file edits.
- Editing source files is a separate, deliberate action (with the normal edit tools). The running kernel picks up edits to physical source files automatically; the next call can check them.
- Do not rely on an in-memory experiment as if it were saved: after a successful experiment, make the real edit.
- A leftover in-memory redefinition can mask or mimic the behavior of the physical source. If a result contradicts the code, reset the kernel and re-check.

## Procedure

1. Run a small `import` call for the project code under test.
2. Try the behavior with real inputs; narrow down with further small calls.
3. When useful, sketch a fix as an in-memory redefinition and compare results.
4. Make the physical source edit, then run the REPL again to confirm. If in-memory sketches remain, redefine them back or reset the kernel first so they do not shadow the edited source.
5. Call `agent-repl-mcp__close(project_full_path)` when the agent or subagent is done with the REPL, including after a failure.

## Errors

- A Python exception raised by your experiment (`ValueError`, `ImportError` from your own code, a failing assertion) is normal output. Read it and continue.
- Unexpected MCP or kernel behavior is different: tool errors, a dead or hung kernel, lost state, wrong `project_full_path` handling, or results that still contradict the code after a kernel reset (which rules out a leftover in-memory redefinition). Report it to the developer promptly with the call made, the expected result versus the actual result or error, and no secrets. Do not silently work around it (for example by switching to `python -c` or a project copy) without saying so.

## Limits

- The REPL does not replace lasting regression tests; add or update tests in the project when behavior changes.
- It does not replace the required lint and unit tests, which `qa-check` owns.
