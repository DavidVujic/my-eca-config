---
name: commit
description: Parent-only. Create a conventional commit from staged changes, with a Code Health gate and a message drafted by the commit subagent.
---

# Commit Skill

Run by the parent agent, and only when the user explicitly asks for a commit. All Git commands go through `eca__git`; the `commit` subagent only drafts the message.

## Procedure

1. Run `git diff --cached --stat`. If the output is empty, stop and tell the user: "No staged changes found. Stage files with `git add` before committing." Never stage files.
2. Run `git diff --cached` and keep the output as the raw staged diff. Do not use the branch or working-tree diff.
3. Code Health gate: run the gate per the AI-touched definition in `AGENTS.md` (unknown origin counts as AI-touched): load the `safeguarding-ai-generated-code` skill via `eca__skill` and follow it. Run `codescene__pre_commit_code_health_safeguard` on the repository. It also checks unstaged modified files. On any degradation, stop and report the findings in two groups: staged files (part of this commit) and unstaged files (not part of it). Continue only after the user explicitly accepts the risk or the findings are fixed. If the tool errors, report that and do not treat the gate as passed.
4. Spawn the `commit` subagent (`agents/commit.md`) with the raw staged diff and the user's hint (`$ARGUMENTS`, may be empty). It returns the full message.
5. Commit with `eca__git`, passing the message on stdin with a quoted heredoc (never inside `$(...)`):

```bash
git commit -F - <<'EOF'
<type>(<scope>): <subject>

<body>
EOF
```

   If a line of the message is exactly `EOF`, use another quoted delimiter that no message line equals.

6. Report the short commit hash from the `git commit` output and the full message.

## Rules

- Never auto-stage; commit only what is already staged.
- Never commit past a failing Code Health gate without the user's explicit acceptance.
- Never push as part of this skill.
