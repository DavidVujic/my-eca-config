---
name: "coding-style"
description: "General coding style policy for generated code: restrictive on comments, concise docstrings for modules and functions."
---

# General coding style policy

Applies when writing or changing code, in any language.

## Comments
- Default: no comment. Naming and structure carry the meaning.
- Comment only for a non-obvious *why*: a tradeoff, hazard, or surprising constraint.
- No references to tickets, issues, or PRs.
- No references to other parts of the codebase — and never to test code.
- No restatements of the code, commented-out code, banners, or edit narration.
- If a comment explains *what* the code does, rename or extract instead.

## Docstrings
- Do write them, for modules and for functions.
- Keep them very concise: one line where possible, stating purpose.
