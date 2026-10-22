---
name: git-changes-context
description: Parent-only. Obtain a unified diff against the branch base in branch mode (committed branch changes) or worktree mode (committed, staged and unstaged tracked changes), using a fixed base order, and report the exact command that produced it.
---

# Git Changes Context Skill

Provide a change set (unified diff) for review, summary, or PR drafting. Only the parent agent runs this skill; subagents have no `eca__git` and never run Git through the shell. The parent passes the result to subagents unmodified.

## Mode

The caller picks the mode; do not switch modes on your own.

- `branch`: commits on `HEAD` since the base. Used for change summaries and PRs.
- `worktree`: everything tracked that differs from the base: committed, staged, and unstaged. Used for `/review` of current changes.

## Procedure

Bases, in this order: `origin/HEAD`, `origin/main`, `origin/master`, `main`, `master`.

Run the mode's command with `eca__git` for each base in order and stop at the first that succeeds with non-empty output:

- `branch`: `git diff <base>...HEAD`
- `worktree`: `git diff --merge-base <base>`

`--merge-base` compares the working tree to the merge base, so upstream commits not in `HEAD` do not appear as reverted changes.

Try every base before giving up; an error or empty output means "next base", not "no diff". Never guess a base outside this list.

In `worktree` mode, also run `git status --short --branch` and collect the `??` entries. `git diff` excludes untracked files, so they are not part of the change set.

## Result

Report to the caller:

- `mode`: `branch` or `worktree`.
- `diff_source_command`: the exact command that produced the diff.
- `diff_context`: the raw unified diff, unmodified.
- `untracked`: `worktree` mode only; untracked paths, or empty. Disclose them to the user as not reviewed.
- `error`: `NO_DIFF_FOUND` when every base errored or returned nothing; otherwise none. Then report `scope`: the mode, that all five bases were tried, and whether any base resolved. In `worktree` mode, still report `untracked`.

Never fabricate or summarize the diff here; consumers do that.
