---
name: "coding-style"
description: "General coding style policy for generated code: restrictive on comments, concise docstrings for modules and functions."
---

# General coding style policy

Applies when writing or changing code, in any language.

## Comments
- Default: no comment. Naming and structure carry the meaning.
- Comment only for a non-obvious *why*: a tradeoff, hazard, or surprising constraint in a very concise way.
- No references to tickets, issues, or PRs.
- No references to other parts of the codebase, other services or test code.
- No restatements of the code, commented-out code, banners, or edit narration.
- If a comment explains *what* the code does, rename or extract instead.

## Docstrings
- Do write them, for modules and for functions.
- Keep them very concise: one line where possible, stating purpose.

## Blank lines inside functions
- Separate logical parts of a function body with one blank line: setup/assignments, validation or control flow, and the return.
- Use it where it helps scanning; short functions of two or three lines need none.
- Do not put blank lines between every statement, and never more than one in a row.
- The language formatter and linter win on any conflict.

## Keep code focused
- Write a small one-off set of fields directly. Use a helper when the logic repeats or becomes complex.
- Handle errors where they occur. For example, keep rendering errors separate from file-writing errors.

## Keep tests focused
- Add tests sparingly: only the smallest set needed to cover distinct behavior.
- Prefer lower-level unit tests next to the component that owns the behavior. Add one only for a meaningful, distinct scenario, boundary, or failure mode.
- Keep higher-level and orchestrator tests narrow and few: coordination, failure handling at that level, and a few representative paths. Do not repeat lower-level scenarios there.
- Keep each test simple and concise: minimal setup, one clear behavior, one clear expected result. Avoid elaborate fixtures, excessive mocking, and verbose assertions.
- Keep a test when it covers a distinct requirement or known regression. Remove or combine tests that duplicate each other, at the same layer or across layers, without adding a new assertion.
