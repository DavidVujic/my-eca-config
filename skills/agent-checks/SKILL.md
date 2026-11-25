---
name: agent-checks
description: Use for lint, typecheck and test through the agent-checks-mcp server, during implementation (with a background watcher) or on a direct check request. Assumes the server is already configured in the agent harness. Covers the stable project_full_path and session_id, watcher startup, reuse and cleanup, polling running, reading origin, unavailable checks, and reporting.
---

# Agent Checks Skill

The `agent-checks-mcp` server runs lint, typecheck and test for a project. It detects the commands itself (the major Python linting and type checking tools, and pytest), and can watch a project in the background so results are ready when you ask instead of waiting for each run. Every tool takes a `project_full_path` and a `session_id`.

## When to use

- During implementation, to keep lint, typecheck and test results up to date after edits, via a background watcher.
- On a direct request to check the code (lint, typecheck, tests), on demand without a watcher.
- Never run lint, typecheck or tests via shell commands; always go through the server, and never substitute your own commands.

## Sessions

- `session_id` is any non-empty string, stable for the whole checks session and unique among concurrent sessions. Prefer the host's chat or session ID; otherwise generate a UUIDv4 once with a real generator. Never fabricate one from the project or task.
- `project_full_path` is the known absolute project root. Pass both unchanged to every checks call; never guess, alter, or fabricate either.
- A sequential session may reuse an id only after the previous watcher stopped. If either value is missing, call no checks tools; report the session as not started and continue.

## Watching during implementation

1. Call `start_watching(project_full_path, session_id)` before the first edit. It returns at once with the detected commands while the initial lint, typecheck and test runs continue in the background. Reuse an active watcher with the same values; restart a stopped or lost one. If startup fails, report the error and continue — the final checks still run on demand.
2. After the last edit, call `lint`, `typecheck` and `test` (all three).
3. Always call `stop_watching(project_full_path, session_id)` after the final results, even when checks fail; on early exit (rejected edit, blocker), stop it as a best effort.

## Direct check requests

Call only the requested checks, or all three for a general request, on demand — without `start_watching` or `stop_watching`.

## Reading results

- Each result has a `running` flag covering only that check kind: while a run of the kind is scheduled or in progress, the tool returns the previous result (or a placeholder without results) with `running: true`. `running: true` is not a current pass, even with an older cached result; call again until `running: false`, or report the check unavailable.
- Each result has an `origin`: `"watcher"`, `"initial"`, or `"on-demand"`, or `"none"` when no commands of that kind were detected.
- A detected check kind has two command scopes: `all` (the full command, the first detection hit) and `files` (a file-scoped variant, when a recipe command without file arguments was backed by a deeper tool that takes file paths). Initial and on-demand runs use the `all` scope; watcher batches use the `files` scope when it was detected, otherwise `all`. Files from the kind's previous run follow the upcoming watcher batches, so the reported changed files cover all files changed so far.
- Each entry has a `scope`: `"all"` or `"files"`, showing which command scope ran; the `argv` in its results shows the exact command, with any changed or affected files appended.
- Pass only for a successful result produced after your last change. Report completed failures as fail, with the failures.
- Errors, `origin: "none"`, no detected command, or a failed or cancelled run without results count as unavailable, never passed.

## Reporting

- Startup (watcher only): started, reused, not started, or failed with the error.
- Per check: pass, fail with failures, or unavailable with the reason, each with its `origin`.
- Cleanup (watcher only): stopped, or the `stop_watching` error.

## Limits

- If no command is detected for a check kind, that check is unavailable — report it, don't improvise a command.
- The checks server runs only the detected lint, typecheck and test commands; it does not answer behavior questions.
