---
mode: subagent
description: Create a pull request for the current feature branch.
model: anthropic/claude-sonnet-4-6
---
# Pull request playbook

Use this playbook whenever the user asks to create a pull request.

## Preconditions

- Branch: run `git branch --show-current` with `eca__git`. If it is `main` or `master`, stop and report `{"error":"refusing_to_pr_from_default_branch"}`.
- Diff: load and execute the `git-changes-context` skill to obtain `diff_context`. If it returns `{"error":"NO_DIFF_FOUND"}`, output `{"status":"no_changes"}` and stop.
- Code Health gate: run `codescene__analyze_change_set` with `base_ref` set to the left side of `...` in the skill's `diff_source_command` (e.g. `origin/main`). If it reports a degradation, stop and report the findings instead of opening the PR; the user decides whether to refactor first or accept the risk explicitly.

## PR content

Derive everything from `diff_context`; do not assume intent beyond what the changes show.

Summary: at most three sentences, one when possible, on what changed and why — not which files or functions were touched. Undesired: "Updated login.js, added a function to handle tokens, and fixed a typo."

Title: Conventional Commit form `<type>(optional-scope): <description>`, derived from the summary. Types by primary intent: `feat` new behavior, `fix` bug fix, `refactor` internal change without behavior change, `perf`, `test`, `docs`, `chore`.

Body: exactly the summary. No headings, file lists, attribution, or tool narration.

## Create

With `eca__git`:

1. `git push -u origin <branch>` if the branch has no upstream.
2. Pass the body on stdin with a quoted heredoc:

```bash
gh pr create --title "<title>" --body-file - <<'EOF'
<summary>
EOF
```

Report the PR URL.
