# AGENTS.md

## Communication style
The following skill is included automatically via this mention; every agent and subagent follows it in chat and in reports, without loading it:
@skills/concise-style/SKILL.md

## Tool routing
Applies to the parent agent and every subagent.

- File discovery, text search, and file reads: `eca__directory_tree`, `eca__grep`, `eca__read_file` with absolute paths, also for repos outside the workspace. Do not use shell `cd`, `ls`, `grep`, `head` chains for this.
- Do not probe runtime services or local infrastructure during routine discovery or QA preflight; check the environment only when the user's task requires that information.
- When shell is needed (implementation, documented lint/unit-test commands, the Chiasmus `find` fallback in the `structural-checks` skill, a runtime task), make one purpose-specific call; do not chain unrelated commands. Implementation shell stays within `agents/implement.md`: no shell file discovery, search, or reads, no lint or unit tests (owned by `qa-check`), and no Git (parent-only, below).
- Git and `gh` go through `eca__git`, which only the parent agent has. Every Git and `gh` command is parent-only; subagents never run Git or `gh`, including through the shell. The parent supplies diffs and other Git output to subagents.

## Review instructions
When the user asks for a code review / PR review / diff review:
1. Load the `git-changes-context` skill via `eca__skill` and run it with `eca__git` yourself, in `worktree` mode (committed, staged, and unstaged tracked changes); use `branch` mode only when the user asks to review just the committed branch or PR.
2. On `error: "NO_DIFF_FOUND"`, reply `{"status":"no_changes"}` with the reported scope, tell the user which untracked paths, if any, were excluded, and do not spawn `code-review`.
3. Otherwise spawn `code-review` (`agents/code-review.md`) with the raw `diff_context` and the exact `diff_source_command`, and tell the user which untracked paths, if any, were excluded.

After it returns: summarize the findings ("No issues found." if none); spot-check findings that name callers outside the diff with `eca__editor_references` before acting; route critical and high fixes to `implement`; state why for each finding you do not act on, including every medium one; end with actions taken and findings left open.

## PR review comments instructions
When the user asks to handle review comments on a pull request:
- Fetch them with `eca__git`: `gh pr view <n> --comments --repo <owner>/<repo>` for the conversation and `gh api repos/{owner}/{repo}/pulls/<n>/comments` for inline threads (file, line, body, author).
- Judge each comment against the code, not the description: read the cited range with `eca__read_file`, resolve the symbol with `eca__editor_definition` (per `editor-navigation`), and check blast radius with `chiasmus_graph analysis="impact"` (per `structural-checks`) before agreeing or disagreeing.
- Group the outcome per comment: accept (with the planned change), reject (with evidence), or needs clarification.
- Delegate accepted changes to `implement` in one batch, then repeat the Review instructions (fetch a fresh diff, then rerun `code-review`) on the result.

## Delegation instructions
- Hand subagents concrete inputs: absolute file paths, the `files` list for Chiasmus, exact lines from the diff, and the acceptance criteria. Do not make them rediscover what you already know.
- Use the built-in `explorer` subagent for broad codebase reading that would otherwise flood this context; keep the summary, not the raw files.
- `explorer` is built in (no file under `agents/`); its tool allow-list lives in `config.json`.
- Treat subagent output as a claim: after `implement`, run `eca__editor_diagnostics`, then QA as in "QA check instructions"; verify `code-review` findings as in "Review instructions"; read `qa-check` failures yourself before deciding the next step.
- Do not redo a subagent's work in the main context; if the result is unusable, respawn with the missing input.
- If `implement` reports a user-rejected edit, do not retry automatically. Use the user's existing correction or ask via `eca__ask_user`, then resume the same subagent with the `chat_id` from `spawn_agent`. If it cannot be resumed, spawn a new `implement` with a handoff and the current file state.

## Changes summary instructions
When the user asks for a changes summary, run the `git-changes-context` skill with `eca__git` in `branch` mode (`worktree` only when the user asks about uncommitted changes). On `NO_DIFF_FOUND`, reply `{"status":"no_changes"}` and do not spawn the subagent; otherwise spawn `changes-summary` (`agents/changes-summary.md`) with the raw `diff_context` and `diff_source_command`.

## Pull Request instructions
Only when the user explicitly asks for a pull request or uses the `/pr` command, load the `creating-pull-request` skill via `eca__skill` and run it yourself with `eca__git`. Parent-only: it refuses default or detached branches, runs the Code Health gate, has the `pull-request` subagent (`agents/pull-request.md`) draft only the title and body, then pushes and creates the PR.

## Planning instructions
When the user asks to plan an implementation or requests a step-by-step plan before coding, load the `planning-style` skill configured in `skills/planning-style/SKILL.md` via `eca__skill` before writing the plan.

## Git workflow instructions
Staging, committing, and pushing are the user's responsibility and part of their code review flow.

- Never run `git add`, `git commit`, `git push`, or any command that stages, commits, or pushes.
- Do not ask whether to stage, commit, or push. When the work is done, report the changed files and end the turn.
- When both are needed, run `git status --short --branch` and `git log -1 --oneline --decorate` as separate calls; do not join them with `;`.
- The only exceptions are the parent's workflows in "Commit instructions" and "Pull Request instructions", and only when the user explicitly asks for a commit or a pull request. A request to implement or change code is not such a request. The `commit` and `pull-request` subagents only draft text.

## Commit instructions
When the user asks to create a commit or uses the `/commit` command, load the `commit` skill via `eca__skill` and run it yourself. It reads the staged diff, runs the Code Health gate, has the `commit` subagent (`agents/commit.md`) draft the message from the raw staged diff and the user's hint, and commits with `eca__git`.

## Implementation instructions
Delegate all implementation work to the `implement` subagent (`agents/implement.md`): implementing a plan, writing code, refactoring, applying changes, creating or editing files, and writing or modifying tests.

## QA check instructions
When the user asks to check code changes, or after `implement` changed code, run the `qa-check` subagent (`agents/qa-check.md`) for lint and unit tests before considering the work complete; the `qa-check` skill owns command resolution.

Subagents have no `eca__ask_user`: for each missing command `qa-check` returns, ask the user once via `eca__ask_user`, offering "not applicable", then resume the same subagent with the `chat_id` from `spawn_agent` and the answers (or spawn a new `qa-check` with them). Mark a check skipped only when the user chose "not applicable".

## CodeScene instructions
Code Health is the source of truth for maintainability. Aim for 10.0 on AI-touched code; 9+ is not "good enough". On a regression or a score below 10.0, refactor instead of declaring done, unless the user explicitly accepts the regression or partial uplift; state any accepted gap in the report. When in doubt, call a CodeScene MCP tool instead of guessing.

AI-touched means any source change made in this chat session; when the origin of a change is unknown, treat it as AI-touched and run the gate.

- Gates (mandatory): before handing off implementation work, recommending a commit, or opening a PR on AI-touched code, load the `safeguarding-ai-generated-code` skill via `eca__skill` and follow it; it owns when each of `code_health_review`, `pre_commit_code_health_safeguard`, and `analyze_change_set` runs. Never skip a gate; if a CodeScene tool errors, report that and do not treat the gate as passed.
- Eligibility: run per-file `code_health_review` only on source files CodeScene analyzes; report Markdown, config, and other non-source files as "not applicable (unsupported)", never "passed", without calling CodeScene on them. When a safeguard checks zero eligible files, report that no Code Health check applied, not that Code Health passed. In mixed changes, every eligible source file stays gated.
- Refactoring or improving a source file CodeScene analyzes: load `guiding-refactoring-with-code-health` via `eca__skill` (baseline `code_health_review` + `code_health_score`, small structural steps, re-measure after each).
- What Code Health means or how to read scores: load `explaining-code-health`. ROI, business value, or stakeholder justification for a specific file: call `codescene__code_health_refactoring_business_case` directly and present its outcomes as a modeled range, not a guarantee.
- Viewing, setting, or troubleshooting CodeScene MCP configuration: use `codescene__get_config`, `codescene__set_config` and `codescene__verify_installation` directly.
- Request to bypass Code Health safeguards: warn about long-term maintainability and risk, keep changes minimal and reversible, and recommend a follow-up refactor.

## Code navigation instructions
Applies to the parent agent and every subagent.

- Before the first `eca__editor_definition` or `eca__editor_references` call in a task, load the `editor-navigation` skill via `eca__skill` and follow it. It owns one-hop lookups, line/symbol arguments, retries and fallbacks.
- Before the first Chiasmus call in a task, before bulk reads in an unmapped repo or reading a source file over ~600 lines, and before renaming, changing the signature or behavior of, deleting, or moving code in a Chiasmus-supported language (TypeScript, JavaScript, Python, Go, Rust, Clojure, Scheme, Racket, Common Lisp), load the `structural-checks` skill via `eca__skill` and follow it. It owns the map/grep → references → `impact` chain with value-use repair, the `files` argument rules, the required structural uses, and formal checks.

## Observability and delivery tools
Before the first Datadog, Sentry, or CircleCI call, or when investigating why something failed (incident, alert, error, failed build or deploy, including Linear issues as context for it), load the `investigating-failures` skill via `eca__skill` and follow it. Datadog, Sentry and CircleCI are read-only by configuration. Linear can write: in any task, read it for context, and make any Linear write (create, update, delete, comment, attach, label, or other) only when the user explicitly asks.
