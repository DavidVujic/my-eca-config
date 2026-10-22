# review

Review the current changes: committed, staged, and unstaged tracked changes against the branch base.

Follow "Review instructions" in `AGENTS.md`. The parent obtains the diff first with the `git-changes-context` skill in `worktree` mode, handles `NO_DIFF_FOUND` and untracked files, and only then spawns `code-review`.

Arguments: $ARGUMENTS
