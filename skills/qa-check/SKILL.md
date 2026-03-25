---
name: qa-check
description: "Run linting and unit tests against the current repository, auto-discovering the correct commands from Justfile, Makefile, pyproject.toml, or package.json."
---

# QA Check Skill

## Purpose

Verify code quality after changes by running **linting** and **unit tests**.
This skill discovers the correct commands for the current repository
automatically, so it works across different project setups and languages.

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

### Step 2 — Identify the project type

Check for project manifest files in `REPO_ROOT`:

- If `pyproject.toml`, `setup.py`, or `setup.cfg` exists → project type is **Python**.
- Else if `package.json` exists → project type is **JS/TS**.
- Else → **unknown**. Report:

  > "Unable to determine project type. Skipping QA checks."

  and **stop** — do not continue to subsequent steps.

---

### Step 3 — Set up the environment

#### Python projects

Check for a virtual environment in this order:

1. `REPO_ROOT/.venv/bin/activate`
2. `REPO_ROOT/venv/bin/activate`

If found, **prefix every shell command** in subsequent steps with:

    source REPO_ROOT/.venv/bin/activate &&

(using the path that was found).

This ensures that `just`, `make`, and direct tool invocations all run
inside the activated virtual environment.

If no virtual environment is found, run commands without a prefix.

#### JS/TS projects

No environment activation is needed. Commands will use the package
manager detected from the project:

- If `REPO_ROOT/pnpm-lock.yaml` exists → use **pnpm**.
- Else if `REPO_ROOT/yarn.lock` exists → use **yarn**.
- Else → use **npm**.

Capture the chosen package manager as `PM` for use in later steps.

---

### Step 4 — Discover the lint command

Try each source below **in order**. Stop at the first match.

#### 4a — Justfile

Check if `REPO_ROOT/justfile` or `REPO_ROOT/Justfile` exists.
If it does, look for a recipe named `lint` (a line starting with `lint`
followed by `:` or arguments).

If found, the lint command is:

    just lint

#### 4b — Makefile

Check if `REPO_ROOT/Makefile` exists.
If it does, look for a target named `lint` (a line starting with `lint:`).

If found, the lint command is:

    make lint

#### 4c — Project manifest fallback

**Python** — inspect `pyproject.toml`:

- Look for a `[tool.ruff]` section → command: `ruff check .`
- Else look for a `[tool.flake8]` or a flake8 dependency → command: `flake8`
- Else look for a `[tool.pylint]` section or a pylint dependency → command: `pylint`
- Else look for a mypy, pyright, or other linter config/dependency and
  use the appropriate command.

**JS/TS** — inspect the `scripts` object in `package.json`:

- If a `lint` script exists → command: `PM run lint`
  (where `PM` is the package manager from Step 3).

#### 4d — No linter found

If none of the above matched, report:

> "No lint command found. Skipping linting."

and continue to Step 6.

---

### Step 5 — Run the lint command

Execute the lint command discovered in Step 4 from `REPO_ROOT`,
using the environment setup from Step 3.

- If the command **succeeds** (exit code 0), report: ✅ Linting passed.
- If the command **fails**, report the full output so the user can see the
  errors, prefixed with: ❌ Linting failed.

Do **not** auto-fix lint errors. Report them and continue to Step 6.

---

### Step 6 — Discover the unit test command

Try each source below **in order**. Stop at the first match.

#### 6a — Justfile

Check the Justfile for a recipe named `test` or `unit-test`
(a line starting with `test` or `unit-test` followed by `:` or arguments).

If found, the test command is:

    just unit-test

or `just test` (prefer `unit-test` if both exist, to avoid
accidentally running integration tests).

#### 6b — Makefile

Check the Makefile for a target named `test` or `unit-test`.

If found, the test command is:

    make unit-test

or `make test` (prefer `unit-test` if both exist).

#### 6c — Project manifest fallback

**Python** — inspect `pyproject.toml`:

- Look for a `[tool.pytest]` or `[tool.pytest.ini_options]` section, or a
  pytest dependency → runner is **pytest**.
- Else fall back to **python -m unittest**.

Exclude integration tests with these heuristics:

- If a `tests/unit` or `tests/unit_tests` directory exists, target it
  directly: `pytest tests/unit` (or equivalent path).
- Otherwise, use a marker/keyword exclusion:
  `pytest -m "not integration"` or `pytest --ignore=tests/integration`.
- If an `integration` directory or `tests/integration` directory exists,
  use `--ignore` to skip it.

**JS/TS** — inspect the `scripts` object in `package.json`:

- If a `test:unit` script exists → command: `PM run test:unit`
- Else if a `test` script exists → command: `PM run test`
  (prefer `test:unit` if both exist, to avoid integration tests).

Where `PM` is the package manager from Step 3.

#### 6d — No test command found

If none of the above matched, report:

> "No unit test command found. Skipping tests."

and continue to Step 8.

---

### Step 7 — Run the unit tests

Execute the test command discovered in Step 6 from `REPO_ROOT`,
using the environment setup from Step 3.

- If the command **succeeds** (exit code 0), report: ✅ Unit tests passed.
- If the command **fails**, report the full output so the user can see the
  failures, prefixed with: ❌ Unit tests failed.

---

### Step 8 — Summary

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
- Prefer the most specific test target (e.g. `unit-test` over `test`,
  `test:unit` over `test`) to avoid accidentally running integration suites.
- Always run commands from `REPO_ROOT`.
- For Python projects, always activate the virtual environment (if detected)
  before running any command — including `just` and `make` targets.
- For JS/TS projects, use the detected package manager (`pnpm`, `yarn`,
  or `npm`) consistently.
- If the project type cannot be determined, stop immediately.
- Do not fabricate command output. Run the actual commands.
