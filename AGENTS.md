<!-- gitnexus:start -->
# GitNexus Required Gates

This project is indexed by GitNexus as **bim-streaming-server**.

## Required Workflow

- Before editing indexed source symbols, run impact analysis for the target symbol and report direct callers, affected execution flows, and risk level.
- If impact analysis returns HIGH or CRITICAL risk, stop and warn the user before editing.
- Before committing or final handoff after code changes, run `gitnexus_detect_changes()` and verify the affected scope matches the intended change.
- For unfamiliar code, prefer GitNexus query/context over broad text search.

## Scope Exceptions

- Docs-only edits do not require GitNexus impact analysis unless they change documented behavior for build, deployment, public APIs, generated code, or operational runbooks.
- Config-only edits do not require GitNexus impact analysis unless they affect build, deployment, runtime behavior, security, migrations, or public interfaces.
- Refactors and symbol renames must still use GitNexus impact/context first; do not use plain find-and-replace for renames.

## Reference

Detailed GitNexus commands, risk levels, resources, and index maintenance notes live in [docs/gitnexus-agent-guide.md](docs/gitnexus-agent-guide.md).

<!-- gitnexus:end -->

## Mission

Maintain this repository as a correct, testable, and maintainable software project.

## Repository Boundary

This repository is the Omniverse Kit BIM streaming server. Keep server runtime, USD/BIM scene loading, rendering, WebRTC server behavior, and server-side command handling here.

Do not place browser viewer application code in this repository. Browser client work belongs in the sibling `web-viewer-sample` repository.

## Codex Stability Defaults

- Use `.codex/config.toml` as the project-level Codex runtime guardrail.
- Keep sandbox writes scoped to this repository root.
- Network access is disabled by default; request approval before dependency downloads or external documentation checks that require live network access.

## Working Rules

1. Work inside this repository only.
2. Do not rewrite large areas unless explicitly requested.
3. Before editing, inspect README.md, AGENTS.md, source tree, tests, and dependency files.
4. Before changes, provide:
   - goal
   - files to touch
   - files not to touch
   - validation commands
5. Implement one concern at a time.
6. Do not mix unrelated changes.
7. Do not delete unknown files.
8. Run relevant tests after changes.
9. Final response must include:
   - changed files
   - summary
   - validation performed
   - remaining risks

## Git / PR Rules

One task should map to one PR.

Do not push or create PR unless explicitly requested.

## Testing Rules

Every feature should include at least one relevant test or smoke check.

A change is not done until validation evidence is reported.
