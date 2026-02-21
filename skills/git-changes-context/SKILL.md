---
name: git-changes-context
description: Reliably obtain a non-empty unified git diff for the current work using safe fallbacks (git diff, origin/master...HEAD, origin/main...HEAD) and return structured JSON containing the diff and the command used.
---

# Git Changes Context Skill

## Purpose

Provide a deterministic way to obtain the actual change set (unified diff)
for use by other agents (code review, security audit, test analysis, etc).

This skill MUST be used whenever a task depends on reviewing changes.

---

## Procedure

Follow these steps strictly and in order.

### Step 1 — Working tree diff

Run:

    git diff

If output is non-empty:
- Set `diff_source_command = "git diff"`
- Set `diff_context = <raw unified diff>`
- Stop.

---

### Step 2 — Compare against origin/master

Run:

    git diff origin/master...HEAD

If output is non-empty:
- Set `diff_source_command = "git diff origin/master...HEAD"`
- Set `diff_context = <raw unified diff>`
- Stop.

---

### Step 3 — Compare against origin/main

Run:

    git diff origin/main...HEAD

If output is non-empty:
- Set `diff_source_command = "git diff origin/main...HEAD"`
- Set `diff_context = <raw unified diff>`
- Stop.

---

## Failure Condition

If ALL of the above produce empty output:

Return:

- `diff_source_command = null`
- `diff_context = ""`
- `error = "NO_DIFF_FOUND"`

Do not guess.
Do not fabricate changes.
Do not continue silently.

---

## Output Contract (STRICT)

You MUST return JSON only.

Return exactly this shape:

{
  "diff_source_command": string | null,
  "diff_context": string,
  "error": string | null
}

Rules:

- No commentary.
- No markdown.
- No explanations.
- No additional keys.
- `diff_context` must contain raw unified diff text.
- If no diff found, `error` must be "NO_DIFF_FOUND".

---

## Guarantees

This skill guarantees:

- Deterministic fallback order
- No hallucinated diffs
- Explicit failure signaling
- Compatibility with strict JSON-only agents
