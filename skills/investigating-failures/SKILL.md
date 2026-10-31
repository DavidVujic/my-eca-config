---
name: investigating-failures
description: Use before any Datadog, Sentry or CircleCI call, and when investigating "why did X fail" (incidents, errors, alerts, failed builds or deploys, with Linear issues as context). Covers read-only limits, approval behavior, which tool owns which question, and the step-by-step investigation flow from evidence to code.
---

# Investigating Failures Skill

## Limits

Datadog, Sentry and CircleCI are read-only by configuration; Linear can write. Approvals depend on the active config and agent scope: calls ask for approval by default unless that config explicitly auto-allows the operation. Make each query specific and batch independent ones in a single turn.

## Lanes

- Datadog: logs, metrics, monitors and monitor groups, incidents, deployment/change events, SLOs. Follow the server's own instruction to load its skill guide before the first query in a domain. `execute_code` and `search_datadog_sdk` are disabled in this config; use the named read tools (`search_*`, `get_*`, `list_*`, `aggregate_*`, `analyze_*`) even though the server instructions prefer `execute_code`.
- Sentry: a specific error — stack trace, breadcrumbs, tags, affected releases, Seer analysis. When both Datadog Error Tracking and Sentry cover the same service, use Sentry for the error's internals and Datadog for its frequency and correlation with deploys.
- CircleCI: failed pipelines and jobs, job logs, test results, flaky tests, artifacts. Use it for CI failures before looking anywhere else.
- Linear: issues and projects — read for context; make any Linear write (create, update, delete, comment, attach, label, or other) only when the user explicitly asks.

## Investigation flow for "why did X fail"

1. Establish when: Datadog monitor state and `get_change_stories`/events around the failure; CircleCI run status if the failure is a build or deploy.
2. Establish what: Datadog logs for the service and window; the Sentry issue if an exception is involved.
3. Locate in code: take the failing function or endpoint from the logs or stack trace and run `chiasmus_graph analysis="path"` from the entry point to it; read only the functions on the path. Use `eca__editor_definition` on the frame's symbol for the exact implementation when available; otherwise open the frame's file and line. Follow the `structural-checks` skill for the Chiasmus calls and `editor-navigation` for the editor calls.
4. Report the chain (evidence → cause → affected callers via `impact`) before proposing a fix. State a cause only as far as the evidence supports it; when the evidence is insufficient, say so, list the candidate causes and the missing evidence, and do not present a guess as the cause. Delegate the fix to `implement` only when the evidence supports the cause.
