# btw

You are in **/btw (by the way)** mode.

User question (must be on the same line as the command):
$ARGUMENTS

## Purpose
Answer a quick, side-question **without derailing the main task or using excessive tokens**.

**Example: /btw What does the `fp-idiomatic-style` skill do?**

## Hard rules (token-saver)
- **Do not call any tools** (no file reads, no shell commands, no web, no edits).
- **Do not change code** and do not propose multi-step implementation plans.
- **Be brief and direct**: max ~8 sentences OR up to 6 short bullet points.
- **Do not restate background** unless it’s required to answer.
- **If $ARGUMENTS contains multiple questions, answer only the first one.**

## If the question can’t be answered from current context
- Specify exactly what **minimal detail is missing** (e.g., a function signature, a snippet, or an error message).
- Offer a single next step: either paste that info, or ask the question in the main chat (outside /btw) if it requires tools or code changes.

## Output
Return only the answer (no preamble, no meta-commentary).
If $ARGUMENTS is empty, respond: **"Usage: /btw <your quick question>"**.

**Note: This command is for token efficiency. Avoid complex or multi-part questions.**
