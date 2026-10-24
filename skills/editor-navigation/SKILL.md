---
name: editor-navigation
description: Use before the first eca__editor_definition or eca__editor_references call in a task. Covers one-hop lookups, line/symbol arguments, retries and fallbacks.
---

# Editor Navigation Skill

Applies to the parent agent and every subagent.

## Routing

- One-hop lookups (definition of X, direct references to X): `eca__editor_definition` / `eca__editor_references`. They ask the editor's language server (or an LSP-like xref backend), so they resolve *exactly* which binding a name refers to; grep and Chiasmus only approximate this.
- Transitive or repo-wide questions (blast radius, can A reach B, dead code, cycles, layering, module clusters): Chiasmus, owned by the `structural-checks` skill. Supported languages: TypeScript, JavaScript, Python, Go, Rust, Clojure, Scheme, Racket, Common Lisp.
- Literal text, comments, strings, non-source files: `eca__grep`.

## Editor navigation

Use `eca__editor_definition` and `eca__editor_references` for:

- The implementation behind a call site already in context → `editor_definition`, instead of grepping the name repo-wide.
- The exact usages to update when changing or renaming a symbol → `editor_references`.

Both need `path`, a **1-based** `line`, and the `symbol` as it appears on that line, so they refine a location you already have rather than discover one:

- Take the line from a file you just read, from `eca__grep output_mode="content"`, or from `chiasmus_map mode="symbol" name=<symbol>`.
- Line numbers shift after edits. Re-derive the line before each call; a stale line fails with "Symbol not found on line N".
- Retry once when the language server is still starting. On any other failure, fall back to `eca__grep` or Chiasmus for that question — do not drop the tool for the rest of the session.

### Editor-specific note (Emacs)

Precision depends on the backend for that language: Python (eglot + `ty`), TypeScript, JavaScript, TSX (tide + tsserver) and Emacs Lisp resolve precisely; Clojure only with a connected CIDER REPL; everything else falls back to dumb-jump regex matching, so treat those results as hints.
