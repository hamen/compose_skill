# Release notes — 4.3.0 (2026-07-06)

**`jetpack-compose-audit` only.** `compose-agent` unchanged at `4.2.1`.

Two additions to the Performance category, plus a scoring-correctness guard. The suite already covered axes 1 (stability/skipping) and 2 (deferred reads) of the recomposition problem; this release adds axis 3 and stops the auditor from crediting no-op "fixes".

## Cross-phase back-writes (axis 3)

A **layout-phase** callback (`onSizeChanged` / `onGloballyPositioned` / `onPlaced`) that writes snapshot state a sibling reads in **composition** creates a measure → write → recompose loop, often per-frame during resize/scroll. The audit already flagged same-body backwards writes; it now catches this cross-phase shape too.

- `search-playbook.md` — new **Cross-Phase Back-Write & Composition Self-Invalidation Heuristic**: gather layout-callback writers, confirm a composition-phase reader, cite both `file:line`.
- `scoring.md` — Performance deduction; promoted to a Critical Finding when the loop is per-frame or spans reused lazy items. A matching Reward bullet credits measured values kept in layout/draw (`Modifier.layout { }` / `Modifier.drawBehind { }`).
- `diagnostics.md` — Quick Triage Recipe step 7 surfaces candidates before rubric reading, scoped to the files that actually hold snapshot collections.

## Composition-phase self-invalidation (related check)

A `SnapshotStateMap` / `SnapshotStateList` (including the `toMutableStateList()` / `toMutableStateMap()` builders) mutated inside a `@Composable` body that also reads it. This is **not** cross-phase — write and read are both in composition — and is **deduped** against the same-body backwards-write rule so it is scored once, never double-counted.

## Do Not Credit — False Leads

A guard table in `scoring.md` Performance: plausible "recomposition fixes" that change nothing, so the auditor neither rewards them nor emits them in `Prioritized Fixes`.

- `remember(index) { … }` around a pure, cheap function of its own key.
- Identity/instance caches for read-only derived maps (use `remember(keys)`).
- Layout modifiers on both the measured and the sibling row (the sibling still reads size in composition).
- `assertRecompositionCount(Exactly(1))` forced on both rows when one correctly recomposes 0 times.
- Under **Strong Skipping**, manually `remember`-ing a callback the compiler already auto-memoizes — that lever only applies SSM-off or on `@NonSkippableComposable` / `@DontMemoize` paths. (Historical note: 4.3.0 also listed "or when the lambda captures an unstable value" — **corrected in 4.3.1**; SSM memoizes lambdas even with unstable captures.)

The guard verifies against runtime recomposition counts / compiler reports rather than the presence of a pattern.

## Evals

`evals/evals.json` cases 12–14 lock in the new behavior: cross-phase layout→composition back-write (12), false leads that must not be credited or suggested under SSM (13), and composition-phase snapshot self-invalidation (14). Audit-only at 4.3.0; `compose-agent` gained matching authoring guidance in 4.3.1.

## Attribution

Axis 3 and the false-lead cases are adapted — reworded and re-cited against `developer.android.com` — from [`chrisbanes/skills`](https://github.com/chrisbanes/skills) `compose-recomposition-performance` (Apache-2.0).

## Versions

`jetpack-compose-audit` → **4.3.0** (SKILL + both `plugin.json`). The 4.2.2 Nav3 work was changelog/content only and never bumped the manifests, so the version string moves from `4.2.0` straight to `4.3.0`.
