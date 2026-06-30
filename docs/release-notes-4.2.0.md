# Compose Skill Suite 4.2.0 Release Notes

Released: 2026-06-17 · shipped in the `v4.2.1` suite release alongside the Navigation 3 decision table.

Compose Skill Suite 4.2.0 adds **Paging 3 in Compose** as a first-class review and audit surface for AI coding agents — using the same thesis as animation 4.0: **LLM tells and guardrails**, not an API encyclopedia.

## What Changed

### `compose-agent` 4.2.0

- Added `skills/compose-agent/references/paging.md` — decision table, golden-path screen recipe, stable `itemKey`, `LoadState` branches, hard nos (`itemSnapshotList`, filter-in-composition), preview pattern. Targets LLM mistakes, not `PagingSource` implementation.
- Added review step #14 for paging screens (`LazyPagingItems`, `collectAsLazyPagingItems`, `LoadState`, stable keys, user-driven refresh/retry).
- Added core instruction and authoring-mode guardrail for paged feeds.
- Cross-linked `performance.md` → `paging.md`.
- Added `paging` discovery keyword to Claude and Cursor manifests.

The new paging reference targets predictable LLM mistakes:

- Using Paging for a bounded in-memory `List` already in a `StateFlow`.
- `items(count = …)` without stable `itemKey` (index keys on paginated feeds).
- Ignoring `LoadState` (blank screens, infinite spinners, silent errors).
- Treating `LazyPagingItems` like a materialized list via `itemSnapshotList`.
- Calling `refresh()` / `retry()` from composition instead of user actions.

**Out of scope:** `PagingSource` / `RemoteMediator` implementation guides.

### `jetpack-compose-audit` 4.2.0

- Keeps the existing four scored categories.
- Does **not** add a separate Paging score. Paging defects map to Performance, State, or Side Effects by root cause.
- Adds **Paging list signals** to the Performance report template (**Paging list correctness**).
- Adds **Paging load-state signals** to the State Management report template (**Paging load-state handling**).
- Extends `search-playbook.md` with a paging-list heuristic.
- Extends `scoring.md` and `canonical-sources.md` with paging-specific rules and official URLs.

## Planning Doc

See [`docs/paging-skill-plan.md`](./paging-skill-plan.md) for scope decisions, audit signal mapping, and the multi-agent validation plan.

## Upgrade

Both skills ship as `4.2.0`. Reinstall or refresh the plugin from this branch / release tag.

Scoped review entry:

```text
compose-agent focus on paging
```

## Validation

The paging guidance was validated through multi-agent cross-review (Claude, Codex, Cursor, Antigravity) against the official Android Paging Compose docs. The validation plan lives in `docs/paging-skill-plan.md`; re-run it and tune `paging.md` / audit heuristics from any disagreements in a follow-up patch.
