# AGENTS.md

## Default Skills
All agents must load the following skill by default:
- `skills/concise-style/SKILL.md`

## Review instructions
When the user asks for a code review / PR review / diff review, use the `code-review` subagent configured in `agents/code-review.md`.

## Five Whys instructions
When the user asks for a 5 Whys analysis, use the `five-whys` subagent configured in `agents/five-whys.md`.

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
When the user asks to check the changes of code, use the `qa-check` subagent configured in `agents/qa-check.md` to run linting and unit tests before considering the work complete.

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

When the user wants to view, set, or troubleshoot CodeScene MCP configuration (access token, on-prem URL, default project, SSL bundle, enabled tools), load `configuring-codescene-mcp`.

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

Chiasmus takes an explicit array of absolute file paths; globs are not expanded. Build the list once per session with a shell `find` (exclude `.venv`, `node_modules`, `target`, `dist`, `build`), reuse it in every call, and always pass `cache=true`.

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
