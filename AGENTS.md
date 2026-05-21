# AGENTS.md

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

Prefer the `chiasmus` MCP tools for any **structural** question about source code (TypeScript, JavaScript, Python, Go, Clojure). Grep only finds string matches; chiasmus answers reachability, impact, and dead-code questions that grep cannot.

Before opening files in bulk or chaining greps, consider:

- **"What's in this repo / what does this file expose / where is X defined?"** → `chiasmus_map` (modes: `overview`, `file`, `symbol`).
- **"What calls X / what does X call / blast radius if I change X?"** → `chiasmus_graph` with `analysis="callers" | "callees" | "impact"`.
- **"Can A reach B? What's the call chain?"** → `chiasmus_graph` with `analysis="reachability"` or `"path"`.
- **"Dead code? Cycles? Layer violations?"** → `chiasmus_graph` with the matching analysis.
- **Full code review of a file set** → `chiasmus_review`, then execute its phases.
- **Formal logic checks** (RBAC conflicts, config consistency, version constraints, state-machine reachability) → `chiasmus_formalize` → fill slots → `chiasmus_verify`. Do not use `chiasmus_solve` or `chiasmus_learn`; both call a remote LLM and are denied here.

When `chiasmus_graph` or `chiasmus_map` would answer a question in one call, use it instead of multiple `eca__grep` rounds. Fall back to `eca__grep` for literal text matches, comments, non-supported languages, or non-source files.

Pass `cache=true` on `chiasmus_graph` and `chiasmus_map` when running more than one analysis over the same files in a session.
