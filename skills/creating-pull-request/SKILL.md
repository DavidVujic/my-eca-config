---
name: creating-pull-request
description: Parent-only. Create a pull request for the current feature branch when the user explicitly asks for one or uses the /pr command, with a default-branch refusal, branch diff, Code Health gate, a title and body drafted by the pull-request subagent, input validation, and a quoted-heredoc `gh pr create`.
---

# Creating Pull Request Skill

Run by the parent agent, and only when the user explicitly asks for a pull request (the `/pr` command counts as such a request). All Git and `gh` commands go through `eca__git`; the `pull-request` subagent only drafts the title and body.

## Procedure

1. Run `git status --short --branch`. If the branch is `main`, `master`, or detached, stop and report `{"error":"refusing_to_pr_from_default_branch"}`. Note any uncommitted or untracked changes as not part of the PR.
2. Run the `git-changes-context` skill in `branch` mode. On `NO_DIFF_FOUND`, reply `{"status":"no_changes"}` and stop.
3. Code Health gate: for AI-touched code, load the `safeguarding-ai-generated-code` skill via `eca__skill` first. Run `codescene__analyze_change_set` with `base_ref` set to the left side of `...` in `diff_source_command`. On a degradation, stop and report the findings; the user decides whether to refactor first or accept the risk explicitly.
4. Spawn `pull-request` (`agents/pull-request.md`) with the raw `diff_context` and `diff_source_command`; it returns the title and body.
5. Validate the inputs before any `git push` or `gh pr create`. Subagent text must not be parsed by the shell:
   - Branch: stop unless it matches `^[A-Za-z0-9._/-]+$`.
   - Title: one line with no control characters, else stop; single-quote it, writing each `'` as `'\''`.
   - Body: keep it verbatim; if a line of it is exactly `EOF`, use another quoted delimiter that no body line equals.
6. If the status line shows no upstream or `ahead`, run `git push -u origin '<branch>'`.
7. Create the PR with the validated title and the body on stdin via a quoted heredoc, then report the PR URL:

```bash
gh pr create --title '<title>' --body-file - <<'EOF'
<body>
EOF
```
