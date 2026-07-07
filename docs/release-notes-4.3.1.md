# Release notes — 4.3.1 (2026-07-06)

**`compose-agent` authoring guidance + a suite-wide Strong Skipping correctness fix** (`jetpack-compose-audit` patched too).

Follow-up to 4.3.0. That release taught the **audit** to catch cross-phase back-writes and no-op "recomposition fixes"; this release teaches the **authoring** skill to avoid writing them in the first place. It closes the drift the 4.3.0 cross-review flagged: a `compose-agent` review could still recommend a pattern the audit rejects.

## `performance.md`

- **Never Back-Write Across Phases** — the reverse of "defer reads". Two shapes, each with good/bad Kotlin:
  - Layout → composition: `onSizeChanged` / `onGloballyPositioned` / `onPlaced` writing state a sibling reads in composition. Fix: consume the measured value in layout/draw (`Layout` / `Modifier.layout { }`), or use `BoxWithConstraints` / `SubcomposeLayout` for genuine *parent* constraints — never round-trip a sibling's measured size through composition-read state.
  - Snapshot-collection mutation in a `@Composable` body: `mutableStateListOf` / `mutableStateMapOf` (or `toMutableState*`) mutated and read in the same composition. Fix: derive from inputs with `remember(keys)`, or mutate from an event / `LaunchedEffect` / state holder.
- **Optimizations That Do Nothing** — the false leads, so the agent stops shipping them: `remember(index)` on a pure function, `remember`-ing an already auto-memoized callback under Strong Skipping, identity caches for read-only derived maps, hoisting without stabilizing captures. Prove a win with recomposition counts / compiler reports, not the presence of a pattern.
- **Grep triggers** extended: `onSizeChanged` / `onGloballyPositioned` / `onPlaced`, and `mutableStateListOf` / `mutableStateMapOf` / `toMutableStateList` / `toMutableStateMap`.

## `SKILL.md`

- New Core Instruction: no cross-phase back-writes, no no-op optimizations.
- Review Process step 4 now names cross-phase back-writes.
- New review-checklist line: does any layout callback or snapshot-collection mutation write state read back in composition?

## `jetpack-compose-audit` — Strong Skipping correctness fix

The 4.3.0 false-lead guard said manual callback `remember` could still matter "when the lambda captures an unstable value." That contradicts the official [Strong Skipping docs](https://developer.android.com/develop/ui/compose/performance/stability/strongskipping): SSM memoizes lambdas **even with unstable captures**, so the carve-out could still bless the exact no-op the guard forbids. Removed from `scoring.md` in all four places; the only remaining exception is SSM-off or a `@DontMemoize` / `@NonSkippableComposable` path.

The same correction lands in `compose-agent/performance.md`: the *Strong Skipping baseline*, *Lambdas In Composables*, and *Optimizations That Do Nothing* sections now agree. The old "lambdas passed across module boundaries to generic composables can miss memoization" bullet was dropped — under SSM, lambda memoization happens at the caller's compilation regardless of the callee's module/generics, so that case is obsolete; the real exceptions are SSM-off / `@DontMemoize` / `@NonSkippableComposable`.

## Eval coverage

The authoring rules share the audit acceptance evals (`jetpack-compose-audit/evals/evals.json` cases 12–14) — `compose-agent` mirrors the same rubric. A dedicated compose-agent *write-mode* eval harness is a tracked follow-up.

## Attribution

Continues the 4.3.0 adaptation from [`chrisbanes/skills`](https://github.com/chrisbanes/skills) `compose-recomposition-performance` (Apache-2.0), cited against `developer.android.com`.

## Versions

`compose-agent` → **4.3.1** and `jetpack-compose-audit` → **4.3.1** (SKILL + both `plugin.json` each).
