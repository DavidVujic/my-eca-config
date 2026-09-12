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

## Chiasmus instructions

Chiasmus answers structural questions about code (calls, reachability, impact, cycles, dead code, module clusters) that neither grep nor the LSP can answer in one step. Supported languages: TypeScript, JavaScript, Python, Go, Rust, Clojure, Scheme, Racket, Common Lisp.

### Tool lanes

- One-hop lookups (definition of X, direct references to X): `eca__editor_definition` / `eca__editor_references`.
- Transitive or repo-wide questions (blast radius, can A reach B, dead code, cycles, layering, module clusters): Chiasmus.
- Literal text, comments, strings, non-source files: `eca__grep`.

### Building the `files` argument

Chiasmus takes an explicit array of absolute file paths; globs are not expanded. Build the list once per session with a shell `find` (exclude `.venv`, `node_modules`, `target`, `dist`, `build`), reuse it in every call, and always pass `cache=true`.

### Required uses

- Task start in an unmapped repo: `chiasmus_map` (mode `overview`) before bulk `eca__read_file`. For large repos, add `chiasmus_graph analysis="hubs"` and `"communities"`.
- Before renaming or changing a function's signature or behavior: `chiasmus_graph analysis="impact" target=<fn>`. Report the affected callers.
- Before deleting a function or module: `analysis="callers"`; confirm with `analysis="dead-code"`.
- After moving code between modules: `analysis="cycles"` and `"layer-violation"` on the touched files.
- Bug hunt "how does input reach X": `analysis="path" from=<entry> to=<X>`, then read only the functions on the path.
- Diff / PR review (owned by the `code-review` subagent): `impact` on every changed or removed function. If a snapshot of the base branch exists, run `analysis="diff" against=<base>` first. While working on the base branch, save one with `chiasmus_graph analysis="summary" cache=true save_snapshot=<branch>`; never switch branches to create it.
- Structural review of a file set (not a diff): `chiasmus_review`, then execute its phases.

### Formal checks

RBAC conflicts, config consistency, dependency version constraints, state-machine reachability: `chiasmus_formalize` → fill slots → `chiasmus_lint` → `chiasmus_verify`. When the user provides a Mermaid flowchart or state diagram, pass it directly with `chiasmus_verify solver="prolog" format="mermaid"`. `chiasmus_solve`, `chiasmus_learn`, and `chiasmus_search` are denied (remote LLM / embedding calls).

## WaveScope instructions

Prefer the `wavescope` MCP tools for *intra-file* navigation and triage of large files (>200 lines), and for token-cheap structural previews. WaveScope treats source as a signal (wavelet transforms) to give multi-resolution views, complexity heatmaps, and structural boundaries — it tracks structure, not specific strings. Interpret the JSON bands/peaks/scores it returns; do not attempt the wavelet math yourself.

Lane boundaries (do not let WaveScope override these):

- **Maintainability / technical-debt verdicts** stay with CodeScene — Code Health remains the source of truth (see "CodeScene instructions"). Treat a WaveScope complexity heatmap as a *triage hint* for where to look, not a quality judgment.
- **Cross-file relationships** (calls, reachability, impact, dead code, cycles) stay with Chiasmus (see "Chiasmus instructions"). WaveScope answers "where inside this file," not "what calls this across the repo."
- **Literal text matches** stay with `eca__grep`.

When to use which tool:

- **"Navigate or modify a region in a large file without reading all of it"** → `query_wavelet_context`, centered on your target line. Read the Coarse band for the major section, the Medium band for surrounding signatures, the Fine band for the immediate snippet; jump via peak positions.
- **"Where is the gnarly/bug-prone logic in this file?"** → `get_complexity_heatmap` / `get_entropy_bands`; focus on high-irregularity scores (near 1.0), skim low-entropy boilerplate. Then confirm any debt conclusion with CodeScene `code_health_review`.
- **"Where inside the core files is the dense logic?"** → `get_important_positions` on the hub files from Chiasmus. Chiasmus picks the files; WaveScope picks the lines.

Reach for WaveScope *before* pulling raw file text: it isolates the exact lines you need at a large token saving, preserving context budget.
