# AGENTS.md

## Default Skills
All agents must load the following skill by default:
- `skills/concise-style/SKILL.md`

## Review instructions
When the user asks for a code review / PR review / diff review, use the `code-review` subagent configured in `agents/code-review.md`.

After the subagent returns:
- Summarize the findings; if there are none, say "No issues found."
- Spot-check findings that name callers outside the diff with `eca__editor_references` before acting on them.
- Address critical and high findings; route fixes to the `implement` subagent. For each finding you do not act on, state why.
- End with actions taken and findings left open.

## PR review comments instructions
When the user asks to handle review comments on a pull request:
- Fetch them with `eca__git`: `gh pr view <n> --comments` for the conversation and `gh api repos/{owner}/{repo}/pulls/<n>/comments` for inline threads (file, line, body, author).
- Judge each comment against the code, not the description: read the cited range with `eca__read_file`, resolve the symbol with `eca__editor_definition`, and check blast radius with `chiasmus_graph analysis="impact"` before agreeing or disagreeing.
- Group the outcome per comment: accept (with the planned change), reject (with evidence), or needs clarification.
- Delegate accepted changes to `implement` in one batch, then rerun `code-review` on the result.

## Delegation instructions
- Hand subagents concrete inputs: absolute file paths, the `files` list for Chiasmus, exact lines from the diff, and the acceptance criteria. Do not make them rediscover what you already know.
- Use the built-in `explorer` subagent for broad codebase reading that would otherwise flood this context; keep the summary, not the raw files.
- Treat subagent output as a claim. After `implement`: `eca__editor_diagnostics`, then the `qa-check` subagent when code changed. After `code-review`: the spot-check described above. After `qa-check`: read the failures yourself before deciding the next step.
- Do not redo a subagent's work in the main context; if the result is unusable, respawn with the missing input.

## Changes summary instructions
When the user asks for a changes summary, use the `changes-summary` subagent configured in `agents/changes-summary.md`.

## Pull Request instructions
When the user asks to make a pull request, use the `pull-request` subagent configured in `agents/pull-request.md`.

## Planning instructions
When the user asks to plan an implementation or requests a step-by-step plan before coding, load the `planning-style` skill configured in `skills/planning-style/SKILL.md` via `eca__skill` before writing the plan.

## Git workflow instructions
Staging, committing, and pushing are the user's responsibility and part of their code review flow.

- Never run `git add`, `git commit`, `git push`, or any command that stages, commits, or pushes.
- Do not ask whether to stage, commit, or push. When the work is done, report the changed files and end the turn.
- The only exceptions are the `commit` and `pull-request` subagents, and only when the user explicitly asks for a commit or a pull request.

## Commit instructions
When the user asks to create a commit or uses the `/commit` command, use the `commit` subagent configured in `agents/commit.md`.

## Implementation instructions
When the user asks to implement a plan, write code, refactor, or apply changes, use the `implement` subagent configured in `agents/implement.md`.

## QA check instructions
When the user asks to check the changes of code, or after the `implement` subagent changed code, use the `qa-check` subagent configured in `agents/qa-check.md` to run linting and unit tests before considering the work complete.

## CodeScene instructions

Treat Code Health as the source of truth for maintainability. Aim for Code Health 10.0 on AI-touched code; 9+ is not "good enough." When Code Health regresses or violates goals, refactor — do not declare done. When in doubt, call a CodeScene MCP tool instead of guessing.

### Safeguard AI-generated code (mandatory before commit / PR)

Before recommending a commit or opening a PR on AI-touched code, load the `safeguarding-ai-generated-code` skill via `eca__skill` and follow it. It gates changes with `pre_commit_code_health_safeguard` (staged files) and `analyze_change_set` (branch vs base ref), and falls back to `code_health_review` on any regression.

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

1. `chiasmus_map mode="symbol" name=<symbol>` — definition sites with lines, and whether the name is ambiguous.
2. `eca__editor_references` at that line — the exact usages to edit.
3. `chiasmus_graph analysis="impact" target=<symbol>` — the transitive chain to report as blast radius.

- More than one definition in step 1 means `callers`/`impact` merge unrelated functions: edit from the references list and label the chain approximate.
- Chiasmus misses functions passed as values (e.g. `reduce(fn, ...)`), which references catches. An empty `callers` result never proves a function is unused; confirm with `dead-code` and `editor_references`.

### Building the `files` argument

Chiasmus takes an explicit array of absolute file paths; globs are not expanded. Build it with a shell `find` (exclude `.venv`, `node_modules`, `target`, `dist`, `build`) and always pass `cache=true`. The array is re-sent on every call, so scope it to the packages involved in the change; use the whole repo only for analyses that need it (`dead-code`, `cycles`, `communities`, `hubs`, `overview`).

### Required uses

- Task start in an unmapped repo: `chiasmus_map` (mode `overview`) before bulk `eca__read_file`. For large repos, add `chiasmus_graph analysis="hubs"` and `"communities"`.
- Source file over ~600 lines: `chiasmus_map mode="file" path=<file>` for the outline, then `eca__read_file` with `line_offset`/`limit` on the ranges you need.
- Before renaming or changing a function's signature or behavior: `chiasmus_graph analysis="impact" target=<fn>`. Report the affected callers.
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
