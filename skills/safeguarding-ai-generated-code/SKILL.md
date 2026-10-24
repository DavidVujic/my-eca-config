---
name: safeguarding-ai-generated-code
description: Use when AI-generated or AI-modified changes need a Code Health gate before commit, handoff, or pull request.
---

# Safeguarding AI-Generated Code

## Overview

Use Code Health safeguards before declaring AI-touched code ready. The goal is to catch maintainability regressions early and prevent agents from normalizing technical debt.

## When to Use

- The agent changed code and is about to suggest a commit.
- The user asks whether a branch or staged changes are safe to merge.
- The workflow needs a quality gate for AI-generated code.

Do not use this skill for broad refactoring discovery or project-level prioritization.

## Quick Reference

- `code_health_review`: Review each supported AI-modified source file immediately after the change.
- `pre_commit_code_health_safeguard`: Check staged and unstaged modified files before handoff and commit.
- `analyze_change_set`: Check a branch or PR-style change set against a base ref.

## Implementation

1. After each AI modification to a source file CodeScene analyzes, run `code_health_review` on that file. Do not call it on Markdown, config, or other non-source files; report them as "not applicable (unsupported)", never as passed.
2. If the review reports maintainability problems or regression risk, refactor the file in small steps and review it again.
3. Run `pre_commit_code_health_safeguard` before handing off implementation work and before commit-oriented recommendations, as a broader gate across staged and unstaged modified files.
4. Run `analyze_change_set` before PR-oriented recommendations on AI-touched source as a final branch-level gate.
5. If either later gate reports a regression, inspect the affected files with `code_health_review` and keep iterating until the issue is removed or the user explicitly accepts the risk.
6. Target Code Health 10.0 on touched source files; a score below 10.0 counts like a regression for steps 2 and 5.
7. If a CodeScene tool errors (for example an expired token or network failure), report that explicitly; never treat the gate as passed.
8. If a gate checks zero eligible files (`checked-file-count` 0), report "no eligible files; Code Health not applicable", not "passed". In mixed changes, non-source files never exempt the source files from steps 1-6.

## Common Mistakes

- Waiting until commit time to run the first Code Health check.
- Treating safeguard output as optional guidance instead of a release gate.
- Declaring work done after a failing safeguard.
- Jumping straight to broad rewrites instead of inspecting the flagged files first.
- Treating an accepted risk as invisible; call it out explicitly.
- Reporting "passed" for unsupported files or for a gate that checked zero eligible files.
