---
name: commit
description: Create a conventional commit from staged changes, with a diff-derived description body.
---

# Commit Skill

## Purpose

Produce a single, well-formed **conventional commit** from the currently staged
changes. The commit message follows the format:

```
<type>(<scope>): <subject>

<body>
```

This skill MUST be used whenever a task depends on creating a commit.

---

## Procedure

Follow these steps strictly and in order.

### Step 1 — Check for staged changes

Run:

    git diff --cached --stat

If the output is **empty**, stop immediately and inform the user:

> "No staged changes found. Stage files with `git add` before committing."

Do **not** auto-stage files. Do **not** proceed.

---

### Step 2 — Obtain the diff context

Load and execute the `git-changes-context` skill to obtain `diff_context`.

If the skill returns `"error": "NO_DIFF_FOUND"`, fall back to using the staged diff:

    git diff --cached

Capture whichever diff is available as `diff`.

---

### Step 3 — Analyse the diff and build the commit message

Using `diff`, determine:

1. **type** — one of: `feat`, `fix`, `refactor`, `docs`, `style`, `test`, `chore`, `perf`, `ci`, `build`, `revert`.
2. **scope** (optional) — the primary module, file, or area affected. Omit parentheses entirely if no clear scope exists.
3. **subject** — a short imperative summary (lowercase, no period, max ~72 chars).
4. **body** — a concise description (1–3 sentences) of *what* changed and *why*, derived from the diff. Focus on intent, not low-level line changes.

If the user provided a **hint** (passed as `$ARGUMENTS`), use it as the subject line.
Still generate the body from the diff. If the hint already contains a type prefix
(e.g. `fix: correct null check`), honour it and do not override the type.

---

### Step 4 — Execute the commit

Construct the full commit message and run:

    git commit -m "<full message>"

Use a HEREDOC to preserve newlines:

```bash
git commit -m "$(cat <<'EOF'
<type>(<scope>): <subject>

<body>
EOF
)"
```

---

### Step 5 — Confirm

Show the user:
- The commit hash (short form).
- The full commit message that was used.

---

## Rules

- Never fabricate changes not present in the staged diff.
- Never auto-stage files — only commit what is already staged.
- Keep the subject line under 72 characters.
- Use imperative mood in the subject (e.g. "add", not "added" or "adds").
- The body should add value beyond the subject; do not simply repeat it.
- If the diff is trivial (e.g. a single typo fix), the body may be a single sentence.
