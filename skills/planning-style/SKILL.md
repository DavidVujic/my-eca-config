---
name: planning-style
description: "Produce an implementation plan with a functional-leaning, idiomatic style mindset; prefers lightweight data structures over dataclasses/pydantic unless justified."
---

# Plan Skill

Produce an implementation plan with a functional-leaning, idiomatic style mindset; prefers lightweight data structures over dataclasses/pydantic unless justified.

STYLE MINDSET (REQUIRED):
Load the `fp-idiomatic-style` and `coding-style` skills via `eca__skill` before writing the plan.
Apply it as a design constraint (architecture and data flow), not as code-generation.

STRUCTURAL CONTEXT (REQUIRED for languages Chiasmus supports):
`files` = absolute paths from a shell `find` (no globs); pass `cache=true`.
- Run `chiasmus_map` (mode `overview`) if the repo is not mapped this session.
- Run `chiasmus_graph analysis="impact" target=<fn>` for each function the plan changes or removes.

PLANNING OUTPUT:
The plan should include:
- Goal and non-goals
- Proposed functions/modules to add or change (names + responsibilities)
- Blast radius: callers affected by each change, from the `impact` results
- Data flow: inputs → transformations → outputs (prefer pure-ish functions and minimal shared mutation)
- Data representations according to the required style mindset
- Edge cases and validation strategy (only if needed)
- Testing approach (what to test, not full test code)

RULES:
- The Plan itself should be written in a very concise way.
- The Plan should keep things simple and be easy to read.
- Prefer small composable functions over large stateful classes.
- Prefer explicit return values over hidden side effects.
- Do not propose new dataclasses/pydantic schemas unless there is a concrete need (validation, invariants, long-lived domain objects).
- Keep it idiomatic for the language in this repo (Pythonic if Python, etc.).
