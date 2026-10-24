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
Load the `code-navigation` skill via `eca__skill` first. `files` = absolute paths per its "Building the `files` argument" section (whole repo by default); pass `cache=true`.
- Run `chiasmus_map` (mode `overview`) if the repo is not mapped this session.
- For each function the plan changes or removes, follow its "Chaining with editor navigation" section (references, then `impact`, including value uses) for the blast radius.

PLANNING OUTPUT:
The plan should include:
- Goal and non-goals
- Proposed functions/modules to add or change (names + responsibilities)
- Blast radius: callers affected by each change, from the `impact` results
- Data flow: inputs → transformations → outputs
- Data representations, per the loaded style skills
- Edge cases and validation strategy (only if needed)
- Testing approach (what to test, not full test code)

RULES:
- The Plan itself should be written in a very concise way.
- The Plan should keep things simple and be easy to read.
- Design choices follow the loaded `fp-idiomatic-style` and `coding-style` skills; do not restate them.
- Break the plan into ordered, independently mergeable and deployable steps where feasible; state dependencies when a step cannot stand alone.
