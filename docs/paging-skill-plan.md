# Paging reference — implementation plan

Status: **implemented on branch `feat/paging-reference-4.2.0` — Composer 2.5 refinement pass applied.**

Last updated: 2026-06-17.

This document is the plan for extending Compose Skill Suite with Paging 3 in Compose coverage. It follows the same product thesis as `animation.md` and `navigation.md`: **guardrails for LLM mistakes**, not an API encyclopedia.

## Problem statement

When a screen uses `PagingData` / `LazyPagingItems`, agents today have no dedicated reference. They fall back to generic lazy-list guidance and improvise:

- `items(lazyPagingItems.itemCount)` without stable `key`
- ignoring `LoadState` (blank screens, infinite spinners, silent errors)
- treating `LazyPagingItems` like an in-memory `List`
- refresh canRefresh()` / `refresh()` wired from composition instead of user events
- keys from list index or `hashCode()` on paginated, merge-prone streams

These are the same failure modes as lazy-list bugs, but with paging-specific triggers (append load, refresh, placeholder gaps, duplicate IDs across pages).

## Product decisions (locked)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| New audit category? | **No** | Same as animation 4.0 — keeps 0–100 scores comparable |
| Where defects score | Performance (keys, list misuse), State (LoadState), Side effects (refresh in composition) | Root-cause mapping |
| Scope of `paging.md` | Compose UI layer only | No `PagingSource` / RemoteMediator implementation guide |
| Style | LLM tells + guardrails + official links | Matches `animation.md`, not third-party cookbook |
| Version bump | **4.2.0** both skills | New reference + audit hooks = minor release |

## Deliverables

### `compose-agent` 4.2.0

- [x] `skills/compose-agent/references/paging.md`
- [x] Review step #14 in `SKILL.md` (before Kotlin style pass)
- [x] Core instruction + authoring-mode guardrail for paging screens
- [x] References list entry + description / `argument-hint` / plugin keywords
- [x] Cross-link from `performance.md` → `paging.md`

### `jetpack-compose-audit` 4.2.0

- [x] `search-playbook.md` — Paging surface map + paging-list heuristic
- [x] `scoring.md` — paging-specific deductions under existing categories
- [x] `report-template.md` — **Paging list signals** (Performance) + **Paging state signals** (State)
- [x] `SKILL.md` — category focus bullets mention paging where relevant
- [x] `canonical-sources.md` — Paging 3 Compose official URLs

### Repo hygiene

- [x] `docs/release-notes-4.2.0.md`
- [x] README What's new (4.2.0 stub)
- [x] `CHANGELOG.md` entry
- [x] `bin/ci` smoke checks for paging coverage
- [x] Manifest versions → 4.2.0 (Claude + Cursor, both skills)

## `paging.md` outline

1. **When paging vs a normal list** — decision table
2. **Collection** — `collectAsLazyPagingItems()` + lifecycle (`flowWithLifecycle` / ViewModel `cachedIn`)
3. **Rendering** — `items(count = lazyPagingItems.itemCount, key = …)` pattern; never index-only keys
4. **LoadState** — refresh / append / prepend; error + retry; empty vs loading vs content
5. **Refresh** — user-driven `refresh()`; not from composition body
6. **Anti-pattern checklist** — 8–10 LLM tells
7. **Primary sources** — official Paging 3 Compose docs only

## Explicitly out of scope (4.2.0)

- Implementing `PagingSource`, `RemoteMediator`, Room + network offline-first
- Paging unit/integration test suites
- Comparing Paging vs hand-rolled infinite scroll
- AOSP source receipts

Revisit only if user demand or audit false-negative rate justifies it.

## Audit signal mapping

| Smell | Report section | Label |
|-------|----------------|-------|
| Missing / unstable `key` on `LazyPagingItems` | Performance | **Paging list correctness** |
| `indexOf` / index keys on paginated stream | Performance | **Paging list correctness** |
| Ignored `LoadState.Error` / no retry UI | State management | **Paging load-state handling** |
| Spinner never dismissed (refresh stuck) | State management | **Paging load-state handling** |
| `refresh()` / `retry()` from composition | Side effects | (existing side-effect language) |
| Exposing raw `LazyPagingItems` from design-system API | Composable API quality | (existing API-shape language) |

## Composer 2.5 refinement (post-initial PR)

Applied to `paging.md` before multi-LLM convergence testing:

- Decision table moved to top (routing-first, like `animation.md`)
- Single **golden path** (`FeedScreen` + `FeedList`) replacing fragmented sections
- **Hard nos** section: `itemSnapshotList`, filter-in-composition
- **`@Preview`** pattern with `PagingData.from(...)`
- Note that direct `items(lazyPagingItems)` is **deprecated** — use `itemKey` + standard `items(count, key, …)`
- Compacted: pull-to-refresh merged into LoadState; API-shape section folded into checklist
- Cross-refs moved to footer

## Validation plan (post-merge)

Multi-agent / multi-model convergence testing (manual, outside this PR):

1. **Fixture repo** with deliberate paging bugs (no keys, ignored LoadState, composition refresh)
2. Run **compose-agent** review on Claude, Codex, Cursor — expect paging step loaded + cited fixes
3. Run **jetpack-compose-audit** — expect findings under Performance/State, not a fifth category
4. **Negative control** — screen without paging should not hallucinate paging rules
5. Collect disagreements → tune `paging.md` / heuristics in a follow-up patch

Success criteria for 4.2.0 GA:

- ≥3/3 agents flag key + LoadState bugs on fixture
- 0/3 agents recommend implementing RemoteMediator when asked to fix UI only
- Audit report names **Paging list correctness** when keys are missing

## References

- Prior art decision: `docs/animation-skill-notes.md` (Paging listed as strongest next candidate)
- Upstream requests: [`android/skills#11`](https://github.com/android/skills/issues/11), [`android/skills#25`](https://github.com/android/skills/issues/25)
- Official: [Paging 3 with Compose](https://developer.android.com/topic/libraries/architecture/paging/v3-compose)
