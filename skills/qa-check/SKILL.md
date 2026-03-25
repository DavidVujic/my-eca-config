---
name: qa-check
description: "Run linting and unit tests against the current repository, auto-discovering the correct commands from Justfile, Makefile, or pyproject.toml."
---

# QA Check Skill

## Purpose

Verify code quality after changes by running **linting** and **unit tests**.
This skill discovers the correct commands for the current repository
automatically, so it works across different project setups.

This skill MUST be used after an agent produces or modifies code.

---

## Procedure

Follow these steps strictly and in order.

### Step 1 — Locate the repository root

Run:

    git rev-parse --show-toplevel

Capture the output as `REPO_ROOT`. All subsequent file lookups and commands
use `REPO_ROOT` as the working directory.

---

### Step 2 — Detect and activate the virtual environment

Check for a Python virtual environment in this order:

1. `REPO_ROOT/.venv/bin/activate`
2. `REPO_ROOT/venv/bin/activate`

If found, **prefix every shell command** in subsequent steps with:

    source REPO_ROOT/.venv/bin/activate &&

(using the path that was found).

This ensures that `just`, `make`, and direct tool invocations all run
inside the activated virtual environment.

If no virtual environment is found, run commands without a prefix.

---

### Step 3 — Discover the lint command

Try each source below **in order**. Stop at the first match.

#### 3a — Justfile

Check if `REPO_ROOT/justfile` or `REPO_ROOT/Justfile` exists.
If it does, look for a recipe named `lint` (a line starting with `lint`
followed by `:` or arguments).

If found, the lint command is:

    just lint

#### 3b — Makefile

Check if `REPO_ROOT/Makefile` exists.
If it does, look for a target named `lint` (a line starting with `lint:`).

If found, the lint command is:

    make lint

#### 3c — pyproject.toml (Python projects)

Check if `REPO_ROOT/pyproject.toml` exists.
If it does, inspect its contents to determine the linter:

- Look for a `[tool.ruff]` section → linter is **ruff**, command: `ruff check .`
- Else look for a `[tool.flake8]` or a flake8 dependency → linter is **flake8**, command: `flake8`
- Else look for a `[tool.pylint]` section or a pylint dependency → linter is **pylint**, command: `pylint`
- Else look for a mypy, pyright, or other type-checker/linter config or dependency and use the appropriate command.

#### 3d — No linter found

If none of the above matched, report:

> "No lint command found. Skipping linting."

and continue to Step 5.

---

### Step 4 — Run the lint command

Execute the lint command discovered in Step 3 from `REPO_ROOT`,
using the virtual environment prefix from Step 2.

- If the command **succeeds** (exit code 0), report: ✅ Linting passed.
- If the command **fails**, report the full output so the user can see the
  errors, prefixed with: ❌ Linting failed.

Do **not** auto-fix lint errors. Report them and continue to Step 5.

---

### Step 5 — Discover the unit test command

Try each source below **in order**. Stop at the first match.

#### 5a — Justfile

Check the Justfile for a recipe named `test` or `unit-test`
(a line starting with `test` or `unit-test` followed by `:` or arguments).

If found, the test command is:

    just unit-test

or `just test` (prefer `unit-test` if both exist, to avoid
accidentally running integration tests).

#### 5b — Makefile

Check the Makefile for a target named `test` or `unit-test`.

If found, the test command is:

    make unit-test

or `make test` (prefer `unit-test` if both exist).

#### 5c — pyproject.toml (Python projects)

Check `REPO_ROOT/pyproject.toml` to determine the test runner:

- Look for a `[tool.pytest]` or `[tool.pytest.ini_options]` section, or a
  pytest dependency → runner is **pytest**.
- Else fall back to **python -m unittest**.

**Important:** Exclude integration tests. Apply these heuristics:

- If a `tests/unit` or `tests/unit_tests` directory exists, target it
  directly: `pytest tests/unit` (or equivalent path).
- Otherwise, use a marker/keyword exclusion:
  `pytest -m "not integration"` or `pytest --ignore=tests/integration`.
- If an `integration` directory or `tests/integration` directory exists,
  use `--ignore` to skip it.

Example for pytest with a `tests/unit` directory:

    pytest tests/unit

#### 5d — No test command found

If none of the above matched, report:

> "No unit test command found. Skipping tests."

and continue to Step 7.

---

### Step 6 — Run the unit tests

Execute the test command discovered in Step 5 from `REPO_ROOT`,
using the virtual environment prefix from Step 2.

- If the command **succeeds** (exit code 0), report: ✅ Unit tests passed.
- If the command **fails**, report the full output so the user can see the
  failures, prefixed with: ❌ Unit tests failed.

---

### Step 7 — Summary

Present a short summary:

| Check       | Result |
|-------------|--------|
| Lint        | ✅ / ❌ / ⏭️ skipped |
| Unit tests  | ✅ / ❌ / ⏭️ skipped |

If either check failed, advise the user to review the errors before
committing.

---

## Rules

- Never auto-fix lint errors or failing tests — only report.
- Never run integration tests.
- Prefer the most specific test target (e.g. `unit-test` over `test`) to
  avoid accidentally running integration suites.
- Always run commands from `REPO_ROOT`.
- Always activate the virtual environment (if detected) before running
  any command — including `just` and `make` targets.
- Do not fabricate command output. Run the actual commands.
