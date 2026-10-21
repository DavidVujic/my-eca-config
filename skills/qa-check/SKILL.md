---
name: qa-check
description: "Run lint and unit tests. Uses a project QA skill or the commands documented in the project's AGENTS.md/rules; asks the user for any missing command instead of guessing."
---

# QA Check Skill

Run linting and unit tests after code changes and report the result. Never auto-fix; never run integration tests.

## Step 1 — Commands documented by the project

Check these project-side sources in order; the first that yields runnable commands wins for each check (lint, unit tests).

1. A project skill: the available skills include one whose description covers this project's testing, linting or QA (projects ship these in `.agents/skills` or `.eca/skills`). Load it with `eca__skill` and follow it.
2. Project instructions already in context: the project's `AGENTS.md` or `.eca/rules` state how to lint or run unit tests, in any wording.

Rules:
- Use only commands that appear verbatim in the project's text (backticks or a code block). Do not compose a command from prose.
- Prefer the unit-test command over a general test command when both are documented.
- If the sources mention tests but give no runnable command, treat the command as missing.

## Step 2 — Ask for missing commands

For each of lint and unit tests without a documented command, ask the user once for the command via `eca__ask_user` if available; otherwise return the missing command(s) to the parent agent to ask. Offer skipping as an option ("not applicable").

Rules:
- Never treat a missing command as skipped before the user confirms.
- Do not search the repository for commands, and do not guess or compose one (no Justfile, Makefile, `pyproject.toml` or `package.json` probing).
- Run only commands documented verbatim by the project or given explicitly by the user.
- A check the user marks as not applicable is `skipped`.

## Step 3 — Run and report

Run lint, then unit tests, from the repository root. Report:

- `lint`: pass / fail / skipped — with the tool's error output on failure
- `unit tests`: pass / fail / skipped — with the failing test output on failure
- The commands used and their source: project skill, project instructions, or user.

If either check failed, say the errors need review before committing.
