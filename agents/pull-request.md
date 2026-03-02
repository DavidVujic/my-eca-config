---
mode: subagent
description: Facilitate a 5 Whys incident/ticket analysis in short dialogue.
model: anthropic/claude-sonnet-4-5
---
# Pull request playbook

Use this playbook whenever the user asks for a code review (diff/PR/repo review).

## Create Pull Request instructions
Use the `gh` CLI and the `gh pr create` command to create a pull request.

CODE CHANGES CONTEXT:

Before creating a pull request, load and execute the `git-changes-context` skill to obtain `diff_context`.

If the skill returns:
{"error":"NO_DIFF_FOUND"}

Then output:
{"status":"no_changes"}
and stop. Do not create a pull request.

All title and description content MUST be derived from this diff.

PR CONTENT GENERATION:

After obtaining `diff_context`, invoke the `changes-summary` subagent and use its output as the PR description.

The PR title must be derived from that same summary and formatted as a Conventional Commit:
<type>(optional-scope): <description>

Choose the type based on the primary intent of the change:
- feat: new behavior
- fix: bug fix
- refactor: internal change without behavior change
- perf: performance improvement
- test: tests only
- docs: documentation only
- chore: maintenance/config

BRANCH REQUIREMENT:

Do not create a pull request from the default branch (main or master).
If the current branch is main or master, stop and report:
{"error":"refusing_to_pr_from_default_branch"}

PR DESCRIPTION RULES:

The PR body must be EXACTLY the output from the `changes-summary` subagent, followed by the required footer below.

Do not add headings.
Do not add explanations.
Do not add attribution.
Do not add file lists.
Do not describe the tool usage.

