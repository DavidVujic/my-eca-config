---
name: "fp-idiomatic-style"
description: "Coding style policy for generated code: prefer idiomatic language conventions with a functional-leaning approach (pure-ish functions, composability), and prefer lightweight data structures over heavy schema/class abstractions unless clearly justified."
---

# Functional-leaning, idiomatic coding style policy

Applies whenever you write new code OR propose code changes, in any language.

## Priorities (in order)
1) Correctness and clarity first.
2) Idiomatic for the target language (e.g., Pythonic for Python, idiomatic TS for TypeScript).
3) Functional-leaning style when it fits naturally:
   - explicit inputs/outputs
   - minimize side effects and shared mutable state
   - small composable functions
   - declarative transformations over imperative loops where readable

## Data modeling preference (important)
- Prefer lightweight structures for transient data over heavy class/schema abstractions.
- Only introduce richer types when clearly beneficial, e.g.:
  - validation/invariants are required
  - long-lived domain objects
  - meaningful behavior/methods beyond data transport

### Language examples
- **Python:** prefer `dict`/`Mapping`/tuples; avoid `dataclasses` or `pydantic` models unless justified.
- **TypeScript:** prefer plain objects, interfaces, or type aliases; avoid classes unless justified.
- **Other languages:** follow the same principle — reach for the lightest idiomatic container first.

## Don't force FP purity
- Avoid clever chaining, excessive `map`/`filter`, or recursion if it reduces readability.
- Choose the most readable idiomatic solution, then make it functional-leaning where it remains clear.

## When editing existing code
- Prefer the smallest change that satisfies requirements.
- Preserve local conventions unless they conflict with correctness/security.
