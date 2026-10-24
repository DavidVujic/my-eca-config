---
name: code-navigation
description: Use before eca__editor_definition, eca__editor_references or any Chiasmus tool, and before renaming, changing the signature or behavior of, deleting, or moving code. Covers editor navigation, the map/grep → references → impact chain with value-use repair, building the Chiasmus `files` argument, required structural uses, and formal checks.
---

# Code Navigation Skill

Applies to the parent agent and every subagent.

## Routing

- One-hop lookups (definition of X, direct references to X): `eca__editor_definition` / `eca__editor_references`. They ask the editor's language server (or an LSP-like xref backend), so they resolve *exactly* which binding a name refers to; grep and Chiasmus only approximate this.
- Transitive or repo-wide questions (blast radius, can A reach B, dead code, cycles, layering, module clusters): Chiasmus. Supported languages: TypeScript, JavaScript, Python, Go, Rust, Clojure, Scheme, Racket, Common Lisp.
- Literal text, comments, strings, non-source files: `eca__grep`.

## Editor navigation

Use `eca__editor_definition` and `eca__editor_references` for:

- The implementation behind a call site already in context → `editor_definition`, instead of grepping the name repo-wide.
- The exact usages to update when changing or renaming a symbol → `editor_references`.

Both need `path`, a **1-based** `line`, and the `symbol` as it appears on that line, so they refine a location you already have rather than discover one:

- Take the line from a file you just read, from `eca__grep output_mode="content"`, or from `chiasmus_map mode="symbol" name=<symbol>`.
- Line numbers shift after edits. Re-derive the line before each call; a stale line fails with "Symbol not found on line N".
- Retry once when the language server is still starting. On any other failure, fall back to `eca__grep` or Chiasmus for that question — do not drop the tool for the rest of the session.

Precision depends on the backend for that language: Python (eglot + `ty`), TypeScript, JavaScript, TSX (tide + tsserver) and Emacs Lisp resolve precisely; Clojure only with a connected CIDER REPL; everything else falls back to dumb-jump regex matching, so treat those results as hints.

## Chiasmus

### Chaining with editor navigation

Chiasmus matches functions by **name** across the whole file set; editor navigation resolves **bindings**. Combine them for any change to an existing symbol:

1. `chiasmus_map mode="symbol" name=<symbol>` — definition line and whether the name is ambiguous. One `eca__grep` for the name (`output_mode="content"`, whole repo) — split the direct hits into direct calls and value uses (passed as an argument, stored in a dict or list, aliased). Do not trace further by hand; that is `impact`'s job.
2. `eca__editor_references` at the definition line — the exact usages to edit.
3. `chiasmus_graph analysis="impact" target=<symbol>` for the direct calls. A value use creates no edge to the target in Python, Go, or Rust (TS/JS and Clojure see bare identifiers passed to calls or known higher-order forms, nothing else), so for each invisible value use run `impact` on the **enclosing function** that receives and calls it.
4. Report the union as the blast radius, marking which callers came from step 3's repair. Never report a single `impact` result on its own: one missing edge drops every caller above it.

- More than one definition in step 1 means `callers`/`impact` merge unrelated functions: edit from the references list and label the chain approximate.
- An empty `impact` or `callers` result never proves a function is unused; confirm with `dead-code` and `editor_references`.

### Building the `files` argument

Chiasmus takes an explicit array of absolute file paths; globs are not expanded. Build the list in this order:

1. Run `eca__directory_tree` first to gather supported source files (exclude `.venv`, `node_modules`, `target`, `dist`, `build`), and join each entry onto its parent directories to build absolute paths.
2. If it omits a relevant hidden or gitignored directory, call `eca__directory_tree` on that directory directly by absolute path.
3. Use shell `find` only when these steps cannot produce the complete file list Chiasmus needs, and narrow it to the known directory.

Do not use shell `grep`, `cat`, `awk`, `ls`, or shell command chains as substitutes for ECA file discovery, search, or reads. Do not run structural checks on a list known to be incomplete. Always pass `cache=true`. Default to the whole repo: parsing is cached, and a module left out of the array drops every caller above it. Scope down only when the path array itself is too large, and then from evidence, not file names: the modules grep hit in step 1 of "Chaining with editor navigation" plus the entry-point modules (`chiasmus_graph analysis="entry-points"`, or the ones the project's `AGENTS.md` names).

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
