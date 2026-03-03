---
name: git-changes-context
description: Reliably obtain a non-empty unified git diff for the current branch using safe fallbacks (origin/HEAD...HEAD, origin/main...HEAD, origin/master...HEAD) and return structured JSON containing the diff and the command used.
---

# Git Changes Context Skill

## Purpose

Provide a deterministic way to obtain the actual change set (unified diff)
for use by other agents (code review, security audit, test analysis, etc).

This skill MUST be used whenever a task depends on reviewing changes.

---

## Procedure

Follow these steps strictly and in order.

**Important:** You MUST attempt *every* step below until you find a **non-empty** diff.
Do **not** return `NO_DIFF_FOUND` after running only one command.

For each step:
- Run the command exactly as written.
- If the command errors **or** produces empty output, proceed to the next step.
- If output is non-empty, set `diff_source_command` to the exact command string, set `diff_context` to the raw unified diff, and stop.

### Step 1 — Compare against origin/HEAD (preferred default-branch fallback)

Run:

    git diff origin/HEAD...HEAD

---

### Step 2 — Compare against origin/main

Run:

    git diff origin/main...HEAD

---

### Step 3 — Compare against origin/master

Run:

    git diff origin/master...HEAD

---

## Failure Condition

If ALL of the above steps either error **or** produce empty output:

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
