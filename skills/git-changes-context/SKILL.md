---
name: git-changes-context
description: Obtain the unified diff of the current branch against its base, trying remote then local base refs in a fixed order, and report which command produced it.
---

# Git Changes Context Skill

Provide the branch's change set (unified diff) for agents that review, summarize, or describe changes.

## Procedure

Run each command with `eca__git`, in this order, and stop at the first that succeeds with non-empty output:

1. `git diff origin/HEAD...HEAD`
2. `git diff origin/main...HEAD`
3. `git diff origin/master...HEAD`
4. `git diff main...HEAD`
5. `git diff master...HEAD`

Try every step before giving up; an error or empty output means "next step", not "no diff".

## Result

Report three things to the caller:

- `diff_source_command`: the exact command that produced the diff.
- `diff_context`: the raw unified diff, unmodified.
- `error`: `NO_DIFF_FOUND` when all five commands errored or returned nothing; otherwise none.

Never fabricate or summarize the diff here; consumers do that.
