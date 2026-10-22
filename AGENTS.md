# AGENTS.md

## Communication style
The following skill is included automatically via this mention; every agent and subagent follows it in chat and in reports, without loading it:
@skills/concise-style/SKILL.md

## Tool routing
Applies to the parent agent and every subagent.

- File discovery, text search, and file reads: `eca__directory_tree`, `eca__grep`, `eca__read_file` with absolute paths, also for repos outside the workspace. Do not use shell `cd`, `ls`, `grep`, `head` chains for this.
- Do not probe runtime services or local infrastructure during routine discovery or QA preflight; check the environment only when the user's task requires that information.
- When shell is needed (implementation, documented lint/unit-test commands, the Chiasmus `find` fallback, a runtime task), make one purpose-specific call; do not chain unrelated commands.
- Git and `gh` go through `eca__git`, which only the parent agent has. Every Git and `gh` command is parent-only; subagents never run Git or `gh`, including through the shell. The parent supplies diffs and other Git output to subagents.

## Review instructions
When the user asks for a code review / PR review / diff review:
1. Load the `git-changes-context` skill via `eca__skill` and run it with `eca__git` yourself, in `worktree` mode (committed, staged, and unstaged tracked changes). Use `branch` mode only when the user asks to review just the committed branch or PR.
2. If the result is `error: "NO_DIFF_FOUND"`, reply `{"status":"no_changes"}` with the reported scope and do not spawn `code-review`.
3. Otherwise spawn the `code-review` subagent configured in `agents/code-review.md`, passing the raw `diff_context` and the exact `diff_source_command`.
4. Tell the user which untracked files, if any, were excluded from the review.

After the subagent returns:
- Summarize the findings; if there are none, say "No issues found."
- Spot-check findings that name callers outside the diff with `eca__editor_references` before acting on them.
- Address critical and high findings; route fixes to the `implement` subagent. For each finding you do not act on, including every medium one, state why.
- End with actions taken and findings left open.

## PR review comments instructions
When the user asks to handle review comments on a pull request:
- Fetch them with `eca__git`: `gh pr view <n> --comments --repo <owner>/<repo>` for the conversation and `gh api repos/{owner}/{repo}/pulls/<n>/comments` for inline threads (file, line, body, author).
- Judge each comment against the code, not the description: read the cited range with `eca__read_file`, resolve the symbol with `eca__editor_definition`, and check blast radius with `chiasmus_graph analysis="impact"` before agreeing or disagreeing.
- Group the outcome per comment: accept (with the planned change), reject (with evidence), or needs clarification.
- Delegate accepted changes to `implement` in one batch, then repeat the Review instructions (fetch a fresh diff, then rerun `code-review`) on the result.

## Delegation instructions
- Hand subagents concrete inputs: absolute file paths, the `files` list for Chiasmus, exact lines from the diff, and the acceptance criteria. Do not make them rediscover what you already know.
- Use the built-in `explorer` subagent for broad codebase reading that would otherwise flood this context; keep the summary, not the raw files.
- Treat subagent output as a claim. After `implement`: `eca__editor_diagnostics`, then the `qa-check` subagent when code changed. After `code-review`: the spot-check described above. After `qa-check`: handle missing commands as in "QA check instructions", and read the failures yourself before deciding the next step.
- Do not redo a subagent's work in the main context; if the result is unusable, respawn with the missing input.
- If `implement` reports a user-rejected edit, do not retry automatically. Use the user's existing correction or ask via `eca__ask_user`, then resume the same subagent with the `chat_id` from `spawn_agent`. If it cannot be resumed, spawn a new `implement` with a handoff and the current file state.

## Changes summary instructions
When the user asks for a changes summary:
1. Run the `git-changes-context` skill in `branch` mode with `eca__git`; use `worktree` mode only when the user asks about uncommitted changes.
2. If the result is `NO_DIFF_FOUND`, reply `{"status":"no_changes"}` and do not spawn the subagent.
3. Otherwise spawn `changes-summary` (`agents/changes-summary.md`) with the raw `diff_context` and `diff_source_command`.

## Pull Request instructions
Only when the user explicitly asks for a pull request, the parent runs this with `eca__git`:
1. Run `git status --short --branch`. If the branch is `main`, `master`, or detached, stop and report `{"error":"refusing_to_pr_from_default_branch"}`. Note any uncommitted or untracked changes as not part of the PR.
2. Run the `git-changes-context` skill in `branch` mode. On `NO_DIFF_FOUND`, reply `{"status":"no_changes"}` and stop.
3. Code Health gate: for AI-touched code, load the `safeguarding-ai-generated-code` skill via `eca__skill` first. Run `codescene__analyze_change_set` with `base_ref` set to the left side of `...` in `diff_source_command`. On a degradation, stop and report the findings; the user decides whether to refactor first or accept the risk explicitly.
4. Spawn `pull-request` (`agents/pull-request.md`) with the raw `diff_context` and `diff_source_command`; it returns the title and body.
5. If the status line shows no upstream or `ahead`, run `git push -u origin '<branch>'`.
6. Create the PR with the body on stdin via a quoted heredoc, then report the PR URL:

```bash
gh pr create --title '<title>' --body-file - <<'EOF'
<body>
EOF
```

Subagent text must not be parsed by the shell. Before calling `eca__git`:
- Branch: stop unless it matches `^[A-Za-z0-9._/-]+$`.
- Title: one line with no control characters, else stop; single-quote it, writing each `'` as `'\''`.
- Body: keep it verbatim; if a line of it is exactly `EOF`, use another quoted delimiter that no body line equals.

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
All implementation work must be delegated to the `implement` subagent (configured in `agents/implement.md`). Use the `implement` subagent when the user asks to:
- Implement a plan, write code, refactor, or apply changes.
- Create or edit files.
- Write or modify tests.

## QA check instructions
When the user asks to check the changes of code, or after the `implement` subagent changed code, use the `qa-check` subagent configured in `agents/qa-check.md` to run linting and unit tests before considering the work complete. The `qa-check` skill owns command resolution.

If `qa-check` returns missing lint or unit-test commands (subagents have no `eca__ask_user`), ask the user once per missing command via `eca__ask_user`, offering "not applicable". Then resume the same subagent with the `chat_id` from `spawn_agent` and the answers; if it cannot be resumed, spawn a new `qa-check` with the answered commands. Mark a check skipped only when the user chose "not applicable".

## CodeScene instructions

Treat Code Health as the source of truth for maintainability. Aim for Code Health 10.0 on AI-touched code; 9+ is not "good enough." When Code Health regresses or falls short of 10.0, refactor — do not declare done. Accept a regression or a partial uplift only when the user explicitly accepts it, and state it in the report. When in doubt, call a CodeScene MCP tool instead of guessing.

### Safeguard AI-generated code (mandatory at handoff, commit, and PR)

Before handing off implementation work, recommending a commit, or opening a PR on AI-touched code, load the `safeguarding-ai-generated-code` skill via `eca__skill` and follow it. It gates changes with `code_health_review` per modified file, `pre_commit_code_health_safeguard` (staged and unstaged modified files) at handoff and commit, and `analyze_change_set` (branch vs base ref) for PRs. Never skip a gate; if a CodeScene tool errors, report that.

### Guide refactoring

When the user asks to refactor or improve a file, load the `guiding-refactoring-with-code-health` skill via `eca__skill`. It enforces a baseline (`code_health_review` + `code_health_score`), small structural steps, and re-measurement after each step.

### Explanation & education

- When the user asks what Code Health means or how to interpret scores, load `explaining-code-health`.
- When the user asks for ROI, business value, or stakeholder justification, load `making-the-business-case-for-code-health`.

### Configuration

When the user wants to view, set, or troubleshoot CodeScene MCP configuration, use `codescene__get_config`, `codescene__set_config` and `codescene__verify_installation` directly.

### Bypass rule

If asked to bypass Code Health safeguards: warn about long-term maintainability and risk, keep changes minimal and reversible, and recommend a follow-up refactor.

## Editor navigation instructions

`eca__editor_definition` and `eca__editor_references` ask the editor's language server (or an LSP-like xref backend) to resolve a symbol. They are scope- and type-aware, so they answer *exactly* which binding a name refers to — something grep and Chiasmus can only approximate.

Use them for:

- The implementation behind a call site already in context → `editor_definition`, instead of grepping the name repo-wide.
- The exact usages to update when changing or renaming a symbol → `editor_references`.

Both need `path`, a **1-based** `line`, and the `symbol` as it appears on that line, so they refine a location you already have rather than discover one:

- Take the line from a file you just read, from `eca__grep output_mode="content"`, or from `chiasmus_map mode="symbol" name=<symbol>`.
- Line numbers shift after edits. Re-derive the line before each call; a stale line fails with "Symbol not found on line N".
- Retry once when the language server is still starting. On any other failure, fall back to `eca__grep` or Chiasmus for that question — do not drop the tool for the rest of the session.

Precision depends on the backend for that language: Python (eglot + `ty`), TypeScript, JavaScript, TSX (tide + tsserver) and Emacs Lisp resolve precisely; Clojure only with a connected CIDER REPL; everything else falls back to dumb-jump regex matching, so treat those results as hints.

## Chiasmus instructions

Chiasmus answers structural questions about code (calls, reachability, impact, cycles, dead code, module clusters) that neither grep nor editor navigation can answer in one step. Supported languages: TypeScript, JavaScript, Python, Go, Rust, Clojure, Scheme, Racket, Common Lisp.

### Tool lanes

- One-hop lookups (definition of X, direct references to X): `eca__editor_definition` / `eca__editor_references`.
- Transitive or repo-wide questions (blast radius, can A reach B, dead code, cycles, layering, module clusters): Chiasmus.
- Literal text, comments, strings, non-source files: `eca__grep`.

### Chaining with editor navigation

Chiasmus matches functions by **name** across the whole file set; editor navigation resolves **bindings**. Combine them for any change to an existing symbol:

1. `chiasmus_map mode="symbol" name=<symbol>` — definition line and whether the name is ambiguous. One `eca__grep` for the name (`output_mode="content"`, whole repo) — split the direct hits into direct calls and value uses (passed as an argument, stored in a dict or list, aliased). Do not trace further by hand; that is `impact`'s job.
2. `eca__editor_references` at the definition line — the exact usages to edit.
3. `chiasmus_graph analysis="impact" target=<symbol>` for the direct calls. A value use creates no edge to the target in Python, Go, or Rust (TS/JS and Clojure see bare identifiers passed to calls or known higher-order forms, nothing else), so for each invisible value use run `impact` on the **enclosing function** that receives and calls it.
4. Report the union as the blast radius, marking which callers came from step 3's repair. Never report a single `impact` result on its own: one missing edge drops every caller above it.

- More than one definition in step 1 means `callers`/`impact` merge unrelated functions: edit from the references list and label the chain approximate.
- An empty `impact` or `callers` result never proves a function is unused; confirm with `dead-code` and `editor_references`.

### Building the `files` argument

Chiasmus takes an explicit array of absolute file paths; globs are not expanded. Use `eca__directory_tree` to gather supported source files (exclude `.venv`, `node_modules`, `target`, `dist`, `build`) and join each entry onto its parent directories to build absolute paths. If it omits relevant files (for example hidden or gitignored ones), inspect those directories directly by absolute path, or fall back to shell `find`. Do not run structural checks on a list known to be incomplete. Always pass `cache=true`. Default to the whole repo: parsing is cached, and a module left out of the array drops every caller above it. Scope down only when the path array itself is too large, and then from evidence, not file names: the modules grep hit in step 1 plus the entry-point modules (`chiasmus_graph analysis="entry-points"`, or the ones the project's `AGENTS.md` names).

### Required uses

- Task start in an unmapped repo: `chiasmus_map` (mode `overview`) before bulk `eca__read_file`. For large repos, add `chiasmus_graph analysis="hubs"` and `"communities"`.
- Source file over ~600 lines: `chiasmus_map mode="file" path=<file>` for the outline, then `eca__read_file` with `line_offset`/`limit` on the ranges you need.
- Before renaming or changing a function's signature or behavior: the map/grep → references → `impact` chain from "Chaining with editor navigation", including the value-use repair. Report the affected callers.
- Before deleting a function or module: `analysis="callers"`; confirm with `analysis="dead-code"`.
- After moving code between modules: `analysis="cycles"` and `"layer-violation"` on the touched files.
- Bug hunt "how does input reach X": `analysis="path" from=<entry> to=<X>`, then read only the functions on the path.
- Diff / PR review (owned by the `code-review` subagent): `impact` on every changed or removed function. If a snapshot of the base branch exists, run `analysis="diff" against=<base>` first. While working on the base branch, save one with `chiasmus_graph analysis="summary" cache=true save_snapshot=<branch>`; never switch branches to create it.
- Structural review of a file set (not a diff): `chiasmus_review`, then execute its phases.

### Formal checks

RBAC conflicts, config consistency, dependency version constraints, state-machine reachability: `chiasmus_formalize` → fill slots → `chiasmus_lint` → `chiasmus_verify`. When the user provides a Mermaid flowchart or state diagram, pass it directly with `chiasmus_verify solver="prolog" format="mermaid"`. `chiasmus_solve`, `chiasmus_learn`, and `chiasmus_search` are denied (remote LLM / embedding calls).

## Observability and delivery tools

Datadog, Sentry and CircleCI are read-only by configuration; Linear can write. Every call asks for approval, so make each query specific and batch independent ones in a single turn.

Lanes:

- Datadog: logs, metrics, monitors and monitor groups, incidents, deployment/change events, SLOs. Follow the server's own instruction to load its skill guide before the first query in a domain.
- Sentry: a specific error — stack trace, breadcrumbs, tags, affected releases, Seer analysis. When both Datadog Error Tracking and Sentry cover the same service, use Sentry for the error's internals and Datadog for its frequency and correlation with deploys.
- CircleCI: failed pipelines and jobs, job logs, test results, flaky tests, artifacts. Use it for CI failures before looking anywhere else.
- Linear: issues and projects — read for context, create or update only when the user asks.

Investigation flow for "why did X fail":

1. Establish when: Datadog monitor state and `get_change_stories`/events around the failure; CircleCI run status if the failure is a build or deploy.
2. Establish what: Datadog logs for the service and window; the Sentry issue if an exception is involved.
3. Locate in code: take the failing function or endpoint from the logs or stack trace and run `chiasmus_graph analysis="path"` from the entry point to it; read only the functions on the path. Use `eca__editor_definition` on the frame's symbol for the exact implementation.
4. Report the chain (evidence → cause → affected callers via `impact`) before proposing a fix; delegate the fix to `implement`.
