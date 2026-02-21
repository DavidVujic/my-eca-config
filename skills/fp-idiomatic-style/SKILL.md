---
name: "fp-idiomatic-style"
description: "Coding style policy for generated code: prefer idiomatic language conventions with a functional-leaning approach (pure-ish functions, composability), and prefer lightweight data structures over dataclasses/pydantic unless clearly justified."
---

# Functional-leaning, idiomatic coding style policy

Applies whenever you write new code OR propose code changes.

## Priorities (in order)
1) Correctness and clarity first.
2) Idiomatic for the target language (e.g., Pythonic for Python).
3) Functional-leaning style when it fits naturally:
   - explicit inputs/outputs
   - minimize side effects and shared mutable state
   - small composable functions
   - declarative transformations (comprehensions, `sorted`, `any/all`, `itertools`)

## Data modeling preference (important)
- Prefer lightweight, dynamic structures for transient data: `dict`/`Mapping`, tuples, simple containers.
- Do NOT introduce `dataclasses` or `pydantic`/schema models if a `dict`/`Mapping` or tuple is sufficient.
- Only introduce `dataclass`/schema types when clearly beneficial, e.g.:
  - validation/invariants are required
  - long-lived domain objects
  - meaningful behavior/methods beyond data transport

## Don’t force FP purity
- Avoid clever chaining, excessive `map`/`filter`, or recursion if it reduces readability.
- Choose the most readable idiomatic solution, then make it functional-leaning where it remains clear.

## When editing existing code
- Prefer the smallest change that satisfies requirements.
- Preserve local conventions unless they conflict with correctness/security.
