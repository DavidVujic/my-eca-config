---
name: qa-check
description: "Run lint and unit tests. Uses a project QA skill or the commands documented in the project's AGENTS.md/rules when present; otherwise discovers them in one shell call from Justfile, Makefile, pyproject.toml or package.json."
---

# QA Check Skill

Run linting and unit tests after code changes and report the result. Never auto-fix; never run integration tests.

## Step 1 — Commands documented by the project

Check these project-side sources in order; the first that yields runnable commands wins, then skip to Step 3.

1. A project skill: the available skills list includes one from the project (`.agents/skills` or `.eca/skills`) whose description covers testing, linting or QA. Load it with `eca__skill` and follow it.
2. Project instructions already in context: the project's `AGENTS.md` or `.eca/rules` state how to lint or run unit tests, in any wording.

Rules:
- Use only commands that appear verbatim in the project's text (backticks or a code block). Do not compose a command from prose.
- Prefer the unit-test command over a general test command when both are documented.
- If the sources mention tests but give no runnable command, treat them as absent and continue to Step 2.

## Step 2 — Otherwise, discover in one call

Run exactly this single shell command from the repository root (`git rev-parse --show-toplevel`):

```bash
echo '--- files'; ls justfile Justfile Makefile pyproject.toml setup.py package.json pnpm-lock.yaml yarn.lock .venv/bin/activate venv/bin/activate 2>/dev/null; \
echo '--- recipes'; grep -hE '^(lint|test|unit-test)([[:space:]][^:]*)?:' justfile Justfile Makefile 2>/dev/null; \
echo '--- pyproject'; grep -E '^\[tool\.(ruff|pytest|flake8|pylint)' pyproject.toml 2>/dev/null; \
echo '--- scripts'; python3 -c 'import json;print(json.load(open("package.json")).get("scripts",{}))' 2>/dev/null; \
echo '--- test dirs'; ls -d tests/unit test/unit tests/integration test/integration 2>/dev/null
```

Choose from the output:

- setup prefix: `source .venv/bin/activate &&` or `source venv/bin/activate &&` if listed. JS package manager: `pnpm` if `pnpm-lock.yaml`, `yarn` if `yarn.lock`, else `npm`.
- lint, first match: `just lint` → `make lint` → `ruff check .` (`[tool.ruff]`) → `flake8` → `pylint .` → `<pm> run lint` (script present) → skip.
- unit tests, first match: `just unit-test` → `just test` → `make unit-test` → `make test` → pytest (`[tool.pytest…]`): `pytest tests/unit` or `pytest test/unit` if the dir exists, else `pytest --ignore=<integration dir>` if one exists, else `pytest` → `python -m unittest` (Python project without pytest) → `<pm> run test:unit` → `<pm> run test` → skip.

If `--- files` listed nothing, report "Unable to determine project type" and stop.

## Step 3 — Run and report

Run lint, then unit tests, from the repository root with the setup prefix. Report:

- `lint`: pass / fail / skipped — with the tool's error output on failure
- `unit tests`: pass / fail / skipped — with the failing test output on failure
- The commands used and their source: project skill, project instructions, or discovery.

If either check failed, say the errors need review before committing.
