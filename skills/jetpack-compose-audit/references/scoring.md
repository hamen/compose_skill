# Scoring

## Category Weights

Use these default weights for Android Jetpack Compose app repositories:

| Category | Weight |
|----------|--------|
| Performance | 35% |
| State management | 25% |
| Side effects | 20% |
| Composable API quality | 20% |

Performance carries the heaviest weight in v1 because Compose performance issues are the most common reason teams audit a codebase, and the smells are the most measurable from source alone. For state-heavy apps with little perf-sensitive UI (forms, dashboards, settings), a 30/30/20/20 split is reasonable — apply judgment and document the choice in the report's "Notes And Limits" section.

### N/A vs Low Confidence

Mark a category `N/A` only when its surface area is structurally absent — for example, scoring API quality on a repo with zero shared/reusable components. If the surface area is merely thin, score it with `Low` confidence instead of dropping it. `N/A` should be rare.

### Renormalization

If a category is `N/A`, renormalize the remaining weights so they still sum to 1.0:

`weight_i_new = weight_i / sum(remaining_weights)`

Worked example — Composable API quality is `N/A`, so the remaining weights are Performance (35%), State (25%), Side effects (20%), summing to 80%:

- Performance: `0.35 / 0.80 = 0.4375` → 44%
- State: `0.25 / 0.80 = 0.3125` → 31%
- Side effects: `0.20 / 0.80 = 0.2500` → 25%

State the renormalization in the report.

## Score Bands

| Score | Status | Meaning |
|-------|--------|---------|
| 0-3 | fail | Systemic issues, repeated misuse, or architecture-level risk |
| 4-6 | needs work | Mixed quality, recurring smells, meaningful refactor value |
| 7-8 | solid | Mostly healthy with some targeted fixes needed |
| 9-10 | excellent | Consistently strong patterns, only minor issues |

## Overall Score

Report both:

- per-category scores on a `0-10` scale
- an overall score on a `0-100` scale

Compute:

`overall = weighted_average(category_scores) * 10`

Round to the nearest whole number.

## Confidence

Add a confidence note in the report:

- `High`: enough Compose surface area, multiple representative modules/files read
- `Medium`: some categories based on limited sample size
- `Low`: small repo, partial module access, or sparse Compose surface

Low confidence does not block scoring, but it must be stated clearly.

## Category Rubric

Each rule below carries an inline citation. **Every deduction in the report must reference the same citation** so readers can verify against the official source.

### Performance

Reward:

- expensive calculations cached with `remember(keys)` or moved out of composition → [docs](https://developer.android.com/develop/ui/compose/performance/bestpractices)
- stable `key =` in lazy layouts where list identity matters → [docs](https://developer.android.com/develop/ui/compose/lists)
- `contentType` on heterogeneous lazy lists, so Compose can reuse compositions only between items of the same type → [docs](https://developer.android.com/develop/ui/compose/lists)
- `derivedStateOf` used for state that changes faster than its observable output (e.g. scroll position → "show button" boolean) → [docs](https://developer.android.com/develop/ui/compose/side-effects)
- deferred reads via lambda modifiers (`Modifier.offset { … }`, `Modifier.graphicsLayer { … }`, `Modifier.drawBehind { … }`) → [docs](https://developer.android.com/develop/ui/compose/performance/bestpractices), [phases](https://developer.android.com/develop/ui/compose/performance/phases)
- absence of backwards writes (writing to state that has already been read in the same composition) → [docs](https://developer.android.com/develop/ui/compose/performance/bestpractices)
- stability hygiene: `@Stable` / `@Immutable` on data classes used as composable params → [docs](https://developer.android.com/develop/ui/compose/performance/stability)
- `kotlinx.collections.immutable` (`ImmutableList`, `PersistentList`) for collection params — earns a skip when Strong Skipping is off, and under Strong Skipping still pays off via stable `equals()` / `hashCode()` (making the instance-equality check cheap and correct) and structural sharing (no per-recomposition `List.copy()`). Reward independently of the compiler track → [stability](https://developer.android.com/develop/ui/compose/performance/stability), [fix](https://developer.android.com/develop/ui/compose/performance/stability/fix)
- `compose_compiler_config.conf` used to mark third-party types stable → [fix](https://developer.android.com/develop/ui/compose/performance/stability/fix)
- typed state factories (`mutableIntStateOf`, `mutableLongStateOf`, `mutableFloatStateOf`, `mutableDoubleStateOf`) for primitives instead of boxed `mutableStateOf<Int>` → [state](https://developer.android.com/develop/ui/compose/state)
- `@ReadOnlyComposable` / `@NonRestartableComposable` used deliberately on hot-path helpers → [strong skipping](https://developer.android.com/develop/ui/compose/performance/stability/strongskipping)
- evidence of Strong Skipping Mode awareness (Kotlin 2.0.20+ / Compose Compiler 1.5.4+); opt-outs (`@NonSkippableComposable`, `@DontMemoize`) used only with justification → [strong skipping](https://developer.android.com/develop/ui/compose/performance/stability/strongskipping)
- evidence of performance-aware configuration such as baseline profiles, R8 / minify enabled in release, or `ProfileInstaller` setup → [baseline profiles](https://developer.android.com/develop/ui/compose/performance/baseline-profiles)
- `ReportDrawnWhen { ... }` used to signal first-meaningful-content for accurate TTID/TTFD metrics → [tooling](https://developer.android.com/develop/ui/compose/performance/tooling)
- edge-to-edge opt-in via `enableEdgeToEdge()` (first-party) rather than `accompanist-systemuicontroller` on projects that support it → [lists](https://developer.android.com/develop/ui/compose/lists)
- evidence the team uses Compose Compiler reports / metrics to verify skippability → [tooling](https://developer.android.com/develop/ui/compose/performance/tooling), [diagnose](https://developer.android.com/develop/ui/compose/performance/stability/diagnose)
- animated values read through lambda modifiers (`Modifier.graphicsLayer { ... }`, `Modifier.offset { ... }`, `Modifier.drawBehind { ... }`) so per-frame updates stay in the layout/draw phase and skip recomposition → [phases](https://developer.android.com/develop/ui/compose/performance/phases), [animation](https://developer.android.com/develop/ui/compose/animation/value-based)
- `Animatable` held in `remember { Animatable(...) }` and driven from a `LaunchedEffect` — state survives recomposition, animation is cancelled on dispose → [animation](https://developer.android.com/develop/ui/compose/animation/value-based)
- `AnimatedContent` used when the call site needs custom enter/exit or size-aware content transforms, while `Crossfade` is used for standard fade-only content swaps → [animation](https://developer.android.com/develop/ui/compose/animation/introduction)
- multiple properties that animate off the same state change share one `updateTransition` instead of independent `animate*AsState` calls that can drift out of sync → [animation](https://developer.android.com/develop/ui/compose/animation/value-based#updatetransition)

Deduct for:

- non-trivial work in composable bodies — collection transforms (`filter` / `sortedBy` / `groupBy` / `sumOf`), O(N) string ops (`split`, `lines`, `lineSequence`, `replace`, `format`), `Regex(...)` constructed or executed inline, hashing or non-trivial constructors. **`remember` is not a sufficient fix**: the cached body still runs on the composition thread on first composition and on every key change, which is a frame hitch on screen entry. The correct fix is moving the work to a presenter / state holder / ViewModel. O(1) is not a free pass — object initialisation, lock acquisition, and side-effectful constructors can be expensive even when nominally constant-time → [bestpractices](https://developer.android.com/develop/ui/compose/performance/bestpractices), [phases](https://developer.android.com/develop/ui/compose/performance/phases)
- lazy list items without stable keys when identity/moves matter → [lists](https://developer.android.com/develop/ui/compose/lists)
- heterogeneous lazy lists missing `contentType` → [lists](https://developer.android.com/develop/ui/compose/lists)
- reading rapidly changing state too high in the tree → [phases](https://developer.android.com/develop/ui/compose/performance/phases), [bestpractices](https://developer.android.com/develop/ui/compose/performance/bestpractices)
- frequent-state values passed to non-lambda modifiers when a layout/draw-phase alternative exists → [bestpractices](https://developer.android.com/develop/ui/compose/performance/bestpractices)
- backwards writes — writing to state already read in the same composition body → [bestpractices](https://developer.android.com/develop/ui/compose/performance/bestpractices)
- cross-phase back-writes — a later phase writing snapshot state an earlier phase already read: a layout-phase callback (`onSizeChanged` / `onGloballyPositioned` / `onPlaced`) writing state that a sibling reads in composition, or a `SnapshotStateMap` / `SnapshotStateList` mutated (`putAll` / `add` / `clear` / `[k] =`) inside a `@Composable` body that also reads it. Deduct when a composition-phase read of the written state exists (measure → write → recompose loop); promote to a Critical Finding when the loop is per-frame or spans reused lazy items → [phases](https://developer.android.com/develop/ui/compose/performance/phases), [bestpractices](https://developer.android.com/develop/ui/compose/performance/bestpractices)
- repeated broad recomposition smells across screens/components → [stability](https://developer.android.com/develop/ui/compose/performance/stability)
- raw `List`/`Map`/`Set` parameters on widely reused composables when the rest of the codebase has the immutable-collections dependency available — deduct when Strong Skipping is OFF (unstable params block skipping outright), or when Strong Skipping is ON but the collection is rebuilt per recomposition in source (e.g. `listOf(a, b)` / `mapOf(...)` in a composable body, a getter that allocates, or `.toList()` / `.filter { }` on every call). Under Strong Skipping without observable churn, do not deduct → [stability](https://developer.android.com/develop/ui/compose/performance/stability)
- unstable composable params whose `equals()` is expensive, allocating, or semantically broken — for example, a plain `class Foo(...)` with identity equality passed to a reusable composable, a `data class` wrapping a large collection (deep `equals` on every recomposition), or a `data class` with mutable fields (stale skip results). Under Strong Skipping, every recomposition runs `equals()` on each unstable param to decide whether to skip; expensive equality can make "skipping" as costly as recomposing, and broken equality makes skipping silently wrong → [strong skipping](https://developer.android.com/develop/ui/compose/performance/stability/strongskipping), [stability](https://developer.android.com/develop/ui/compose/performance/stability)
- instance-recreation churn in source: `listOf(...)` / `mapOf(...)` / `setOf(...)` / anonymous object or lambda literals / `MyParams(...)` allocated inside a hot composable body and passed as a param. The value is `!=` to the prior recomposition's value by default, so both vanilla skipping and Strong Skipping's `==` gate fail, forcing the callee to re-run body-and-children. Hoist into `remember(...)`, a state holder, or a `@Composable` caller further up → [stability](https://developer.android.com/develop/ui/compose/performance/stability), [strong skipping](https://developer.android.com/develop/ui/compose/performance/stability/strongskipping)
- `mutableStateOf<Int|Long|Float|Double>` where the typed factory exists (autoboxing) → [state](https://developer.android.com/develop/ui/compose/state)
- `derivedStateOf { ... }` whose block does not actually read any `State` object (meaning it will never invalidate, and the overhead of `derivedStateOf` is wasted) → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- `@NonSkippableComposable` / `@DontMemoize` opt-outs without a justifying comment → [strong skipping](https://developer.android.com/develop/ui/compose/performance/stability/strongskipping)
- if the project is on a Compose Compiler track older than 1.5.4 / Kotlin 2.0.20, stability matters more than the rules above assume — note this in the report and weight unstable-param findings more heavily → [strong skipping](https://developer.android.com/develop/ui/compose/performance/stability/strongskipping)
- `remember { … }` whose body reads `LocalConfiguration`, `LocalDensity`, or `LocalLayoutDirection` without declaring those values as keys — the cached value silently goes stale on rotation / foldable posture / font-scale changes → [state](https://developer.android.com/develop/ui/compose/state)
- `indexOf()` / `lastIndexOf()` / `indexOfFirst { }` called inside a `LazyListScope` item factory — O(n²) scroll cost and crash risk when identity moves; use `itemsIndexed` instead → [lists](https://developer.android.com/develop/ui/compose/lists)
- `animateItemPlacement()` on Compose 1.7+ — replaced by `Modifier.animateItem()` → [lists](https://developer.android.com/develop/ui/compose/lists)
- Accompanist libraries where first-party replacements exist: `accompanist-pager` (→ `HorizontalPager`), `accompanist-swiperefresh` (→ `PullToRefreshBox`), `accompanist-flowlayout` (→ `FlowRow` / `FlowColumn`), `accompanist-systemuicontroller` (→ `enableEdgeToEdge()`) — deduct only when the replacement is available on the project's Compose version → [lists](https://developer.android.com/develop/ui/compose/lists)
- `Canvas` / `Spacer` with only `Modifier.fillMaxSize()` and no explicit height or aspect ratio — may enter draw with `Size.Zero`, producing `NaN` math and Skia-pipeline crashes. Require `Modifier.size(...)`, `Modifier.height(...)`, or `Modifier.aspectRatio(...)` on drawing surfaces → [bestpractices](https://developer.android.com/develop/ui/compose/performance/bestpractices)
- lazy-list `key = { ... }` computed from a source that cannot guarantee uniqueness (merged flows, paginated streams, `hashCode()` on non-`data class`) — `IllegalArgumentException: Key ... was already used` crashes production. Verify with the duplicate-lazy-key heuristic in `search-playbook.md` → [lists](https://developer.android.com/develop/ui/compose/lists)
- `Animatable(...)` created inside a composable body without `remember { Animatable(...) }` (and not hoisted from a state holder) — the animation state is rebuilt on recomposition, which restarts the animation and drops in-flight velocity/target → [animation](https://developer.android.com/develop/ui/compose/animation/value-based)
- per-frame animated values read in the composition body and piped into non-lambda modifiers (`Modifier.offset(x.value.dp)`, `Modifier.alpha(alpha.value)`, `Modifier.rotate(rot.value)`). Every frame triggers recomposition of the caller instead of only re-laying-out or re-drawing. Prefer lambda-form `Modifier.offset { IntOffset(x.value.roundToInt(), 0) }` / `Modifier.graphicsLayer { alpha = alphaAnim.value; rotationZ = rot.value }` → [phases](https://developer.android.com/develop/ui/compose/performance/phases), [animation](https://developer.android.com/develop/ui/compose/animation/value-based)
- `rememberInfiniteTransition()` hosted in a composable that stays in composition while offscreen (e.g. inside a `Scaffold` content that does not unmount on tab switch) — the animation keeps running until the host is removed from composition, creating needless offscreen work. Host it inside `AnimatedVisibility` / lazy-list item, or gate with an `if (visible) { ... }` branch → [animation](https://developer.android.com/develop/ui/compose/animation/value-based)
- several `animate*AsState` calls in the same composable all keyed off the same boolean/enum when `updateTransition` would keep them synchronized — light deduction when the drift is user-visible (alpha/scale/color finishing at different times) → [animation](https://developer.android.com/develop/ui/compose/animation/value-based#updatetransition)

When any of these animation rules affects the Performance score, name it in the Performance section as **Animation phase correctness** rather than folding it into a generic recomposition note. Readers should be able to see at a glance whether animation contributed to the score.

- `LazyPagingItems` rendered with `items(count = …)` but **missing `key =`** or using **index-only keys** on a paginated/merged feed — reorder, refresh, and prepend can crash or thrash compositions → [paging compose](https://developer.android.com/topic/libraries/architecture/paging/v3-compose), [lists](https://developer.android.com/develop/ui/compose/lists)
- treating `LazyPagingItems` like a materialized list (`itemSnapshotList` primary UI path, `forEach` into a nested lazy list) instead of the supported `items(count = …)` integration → [paging compose](https://developer.android.com/topic/libraries/architecture/paging/v3-compose)

When any of these paging rules affects the Performance score, name it in the Performance section as **Paging list correctness** rather than folding it into a generic lazy-list note.

Suggested interpretation:

- `9-10`: clean patterns are common and performance smells are rare
- `7-8`: minor or localized issues
- `4-6`: repeated recomposition or lazy-list issues
- `0-3`: serious, widespread performance problems or unsafe state-write patterns

#### Measured Ceilings (apply when compiler reports are available)

Compiler reports generated in Step 4 give hard numbers. When present, apply these ceilings *after* qualitative scoring — a category cannot exceed the cap even if qualitative evidence is strong. Report the applied ceiling in the Performance section so the score is auditable.

Let `skippable%` = `skippableComposables / restartableComposables` from `*-module.json`.
However, because zero-argument lambdas structurally cannot skip and artificially anchor this overall metric, you MUST also compute the **named-only `skippable%`** from `*-composables.csv` (by filtering out rows where `isLambda == "1"`). Use this **named-only percentage** for the ceiling conditions below, and state the distinction clearly in the report.

**Before applying a table, determine whether Strong Skipping is active** (Kotlin **2.0.20+** / Compose Compiler **1.5.4+** is the default; earlier versions can opt in via `-Pandroidx.compose.compiler.enableStrongSkippingMode=true` or the equivalent `freeCompilerArgs` flag). This choice changes which table applies, because Strong Skipping fundamentally changes what `skippable%` and the unstable-classes count are worth.

##### When Strong Skipping is OFF (pre-2.0.20 or explicitly disabled)

Unstable params genuinely block skipping, so both `skippable%` and the unstable-classes count are meaningful signals.

| Condition | Ceiling |
|-----------|---------|
| `skippable%` ≥ 95% AND zero unstable classes used as shared/reusable composable params | no cap (9-10 possible) |
| `skippable%` ≥ 85% AND ≤3 unstable classes used as shared/reusable composable params | cap at 8 |
| `skippable%` 70-85% OR 4-7 unstable classes used as params | cap at 6 |
| `skippable%` 50-70% OR ≥8 unstable classes used as params | cap at 4 |
| `skippable%` < 50% | cap at 3 |
| Strong Skipping disabled on a Kotlin 2.0.20+ project without written justification | cap at 4 |
| `@NonSkippableComposable` / `@DontMemoize` used on hot-path composables without justification | cap at 5 |

##### When Strong Skipping is ON (Kotlin 2.0.20+ default)

Under Strong Skipping, all restartable composables become skippable and lambdas are auto-memoized; skipping is decided per-recomposition via `==` on unstable params. `skippable%` therefore trends toward ~100% almost by construction and loses most of its discriminating power — so the ceiling leans more heavily on qualitative evidence. The unstable-classes *count* also stops mattering on its own; what matters is whether those unstable params are (a) re-allocated per recomposition, or (b) backed by expensive or broken `equals()`.

| Condition | Ceiling |
|-----------|---------|
| Named-only `skippable%` ≥ 95% AND no observed instance-recreation churn AND no expensive / broken `equals()` on unstable params AND no unjustified `@NonSkippableComposable` / `@DontMemoize` | no cap (9-10 possible) |
| Named-only `skippable%` ≥ 95% BUT a handful of unstable params are recreated per recomposition OR carry non-trivial `equals()` cost | cap at 8 |
| Widespread instance-recreation churn in source (e.g. `listOf(...)` / `mapOf(...)` / anonymous literals in hot composable bodies), OR expensive / broken `equals()` on unstable params passed through widely reused composables | cap at 6 |
| Named-only `skippable%` < 90% despite Strong Skipping being on (typically: `@NonSkippableComposable` / `@DontMemoize` spread across hot paths, inline/value-class edge cases, or compiler-rejected composable shapes) | cap at 4 |
| `@NonSkippableComposable` / `@DontMemoize` used on hot-path composables without justification | cap at 5 |

Under Strong Skipping the more informative evidence is no longer "how many unstable classes are there" but:

- **Instance-recreation churn in source** — `listOf(a, b)`, `mapOf(...)`, `{ ... }` object / lambda literals, or `MyParams(a, b)` allocated inside a composable body and passed as a param. The SSM `==` check fails on every recomposition and the callee re-runs regardless of skippability.
- **`equals()` cost and correctness on unstable params** — SSM calls `equals()` on each unstable param every recomposition. Expensive equality (`data class` wrapping a large collection, custom `equals` that walks a graph) can make skipping as costly as recomposing; broken equality (identity equality on a plain `class`, mutable fields inside a `data class`) makes skipping silently wrong.
- **Runtime recomposition counts** (Layout Inspector, `Recomposer` debug output, Compose UI tests with `Modifier.testTag` + recomposition counters). When available these beat `skippable%` as the primary signal under SSM.
- **`@NonSkippableComposable` / `@DontMemoize` opt-outs** — effectively the only way to see a "not skippable" named composable under SSM; each instance should carry a justifying comment and appear on hot paths only with cause.

When compiler reports are **not** available (Step 4 failed, `Compiler diagnostics used: no`), ceilings do not apply — rely on source-inferred judgment, but cap any Performance score at 7 to reflect reduced confidence.

If a non-trivial subset of modules failed to build (partial reports), state which modules contributed and treat `skippable%` as a floor estimate rather than a ground truth.

#### Do Not Credit — False Leads

Changes that *look* like recomposition fixes but do not reduce recomposition count. **Never reward them, and never emit them in `Prioritized Fixes`.** If the codebase already applies one, treat it as neutral (no credit); if a finding's suggested fix would be one of these, discard the finding or replace the fix with the real one. When code applies a false lead *in place of* the real fix, the underlying smell still stands and is scored on its own merits.

| Looks like a fix | Why it does nothing | Real lever |
|---|---|---|
| `remember(index) { isFirstRow(index) }` (or any `remember(k)` wrapping a pure, cheap function of its own key) | Same inputs each call; the memo saves no work and adds bookkeeping — no skipping benefit | Inline the expression; only `remember` genuinely expensive work keyed on real inputs |
| Identity/instance cache for a read-only derived map (`remember { }` returning a cached map to preserve reference equality) | Can serve stale overlays; `remember(keys)` on the actual inputs is enough and correct | `remember(keys) { derive() }` — key on the inputs, not on identity |
| `mutableIntStateOf` + a layout modifier applied to **both** the measured row and its sibling | The sibling still reads the size **in composition** unless it is measure-only; boxing was never the problem | Make the sibling read measured size in layout/draw (`Modifier.layout { }`), or hoist the shared value out of the round-trip |
| Forcing `assertRecompositionCount(Exactly(1))` on **both** rows in a focus-move / selection test | One row often *correctly* recomposes 0 times; the assertion encodes a wrong expectation, not a fix | Assert the count each row should actually have; investigate only rows that exceed it |
| Hoisting state up without stabilizing the lambda captures passed back down | A fresh lambda instance every recomposition still defeats skipping on the child | Hoist **and** stabilize — `remember` the callbacks, or pass method references / stable event objects |

Route: these mirror the search-playbook cross-phase and stability heuristics. Before crediting any "we optimized recomposition here" claim, confirm it is not one of the above — prefer runtime recomposition counts or compiler reports as proof over the presence of the pattern.

### State Management

Reward:

- clear single source of truth → [architecture](https://developer.android.com/develop/ui/compose/architecture)
- correct hoisting to the lowest common reader / highest writer → [state hoisting](https://developer.android.com/develop/ui/compose/state-hoisting)
- related state hoisted together when driven by the same events → [state hoisting](https://developer.android.com/develop/ui/compose/state-hoisting)
- stateless reusable composables with stateful wrappers where useful → [state hoisting](https://developer.android.com/develop/ui/compose/state-hoisting)
- `rememberSaveable` for UI state that should survive recreation → [state](https://developer.android.com/develop/ui/compose/state)
- custom `Saver` / `mapSaver` / `listSaver` / `@Parcelize` for non-bundleable types in `rememberSaveable` → [state](https://developer.android.com/develop/ui/compose/state)
- `collectAsStateWithLifecycle()` in Android UI code → [state](https://developer.android.com/develop/ui/compose/state)
- observable immutable state instead of mutable non-observable containers → [state](https://developer.android.com/develop/ui/compose/state)
- correct observable collections via `mutableStateListOf` / `mutableStateMapOf` instead of wrapping `mutableListOf`/`mutableMapOf` in a `MutableState` → [state](https://developer.android.com/develop/ui/compose/state)
- typed state factories (`mutableIntStateOf` and friends) — cross-listed with Performance because the failure mode is autoboxing → [state](https://developer.android.com/develop/ui/compose/state)
- plain state-holder classes when screen logic grows; idiomatic shape is `@Stable class FooState(...)` paired with a `@Composable fun rememberFooState(...): FooState` factory → [architecture](https://developer.android.com/develop/ui/compose/architecture)
- ViewModel used as the source of truth for screen-level state, scoped at the screen level (not deep in the tree, not inside reusable components) → [architecture](https://developer.android.com/develop/ui/compose/architecture), [state](https://developer.android.com/develop/ui/compose/state)
- ViewModel-exposed flows converted with `.stateIn(scope, SharingStarted.WhileSubscribed(5_000), initial)` — survives configuration changes without restarting upstream work → [architecture](https://developer.android.com/develop/ui/compose/architecture)
- `remember(key)` invalidation when cached values depend on inputs → [state](https://developer.android.com/develop/ui/compose/state)

Deduct for:

- duplicated or split ownership of state → [state hoisting](https://developer.android.com/develop/ui/compose/state-hoisting)
- under-hoisted shared state → [state hoisting](https://developer.android.com/develop/ui/compose/state-hoisting)
- reusable components with unnecessary internal state → [state hoisting](https://developer.android.com/develop/ui/compose/state-hoisting)
- misuse of `remember` where `rememberSaveable` is more appropriate → [state](https://developer.android.com/develop/ui/compose/state)
- non-observable mutable collections or mutable data holders used as state (`mutableListOf` held in a `var`, `ArrayList` mutated in place) → [state](https://developer.android.com/develop/ui/compose/state)
- `mutableListOf` / `mutableMapOf` wrapped in a `MutableState` where `mutableStateListOf` / `mutableStateMapOf` would correctly observe element changes → [state](https://developer.android.com/develop/ui/compose/state)
- Android flows collected in UI without lifecycle awareness when the code is Android-specific (skip on Compose Multiplatform code paths) → [state](https://developer.android.com/develop/ui/compose/state)
- state scattered across multiple sibling composables without a clear owner → [state hoisting](https://developer.android.com/develop/ui/compose/state-hoisting)
- `remember { computeFromInput(x) }` with no `key` — stale cached value when `x` changes → [state](https://developer.android.com/develop/ui/compose/state)
- `viewModel()` invoked deep in a composable tree (rather than at the screen entry point) or ViewModels passed via `CompositionLocal` → [architecture](https://developer.android.com/develop/ui/compose/architecture), [compositionlocal](https://developer.android.com/develop/ui/compose/compositionlocal)
- `rememberSaveable { mutableStateOf(SomeNonBundleable(...)) }` without a `Saver` — restoration silently fails after process death → [state](https://developer.android.com/develop/ui/compose/state)
- string-based navigation routes (`composable("home")`, `navigate("profile/$id")`) on Navigation Compose 2.8+ where type-safe `@Serializable` routes are available — loses compile-time checking and encourages argument-encoding bugs → [navigation](https://developer.android.com/develop/ui/compose/navigation)
- `mutableStateOf` held in a `ViewModel` instead of `StateFlow` / `MutableStateFlow` — couples the ViewModel to the Compose runtime and hurts testability. App-level teams may accept this tradeoff deliberately; note the tradeoff rather than deducting heavily unless it is widespread → [architecture](https://developer.android.com/develop/ui/compose/architecture)
- ViewModel-to-UI events modeled only as `Channel` / `SharedFlow` when the outcome must survive lifecycle gaps (snackbars, validation messages, navigation decisions). Prefer durable UI state with an acknowledgement callback for must-deliver outcomes; reserve `Channel` / `SharedFlow` for ephemeral signals with documented delivery semantics → [UI events](https://developer.android.com/topic/architecture/ui-layer/events), [architecture](https://developer.android.com/develop/ui/compose/architecture)
- `rememberSaveable` used inside a `LazyListScope` item factory (per-item expansion state, per-item form fields) — each entry is serialized into the saved-state `Bundle` which is capped at ~1 MB; large lists trigger `TransactionTooLargeException` → [state](https://developer.android.com/develop/ui/compose/state)
- paging UI that ignores `LazyPagingItems.loadState` — blank screen during initial refresh, stuck spinner, or silent failure with no `retry()` path → [paging compose](https://developer.android.com/topic/libraries/architecture/paging/v3-compose)
- duplicate manual loading flags (`mutableStateOf(isLoading)`) shadowing `loadState` on paging screens — two sources of truth for the same UX → [paging compose](https://developer.android.com/topic/libraries/architecture/paging/v3-compose)
- `loadState.append is LoadState.Error` handled only as a spinner or not at all — append failures leave the user with no way to resume the feed; an append-error footer must offer `retry()` → [paging compose](https://developer.android.com/topic/libraries/architecture/paging/v3-compose)
- `lazyPagingItems[index]!!` (or assuming every slot is non-null) — with `PagingConfig.enablePlaceholders = true`, slots are `null` while a page loads and `!!` NPE-crashes; the dual smell is a placeholder branch that can never run because placeholders are disabled → [paging compose](https://developer.android.com/topic/libraries/architecture/paging/v3-compose)
- `loadState.refresh is LoadState.Error -> FullScreenError()` with **no `itemCount == 0` guard** — a transient pull-to-refresh failure then wipes an already-loaded feed and replaces it with a full-screen error. Full-screen error should gate on `itemCount == 0`; refresh errors arriving with content on screen belong in a snackbar / inline retry → [paging compose](https://developer.android.com/topic/libraries/architecture/paging/v3-compose)

When any of these paging rules affects the State management score, name it in the State section as **Paging load-state handling** rather than folding it into a generic state note.

**Navigation 3 deductions (State management)**

When Nav3 is detected (see playbook section 2b), also deduct for:
- `entryDecorators` list supplied without re-adding `rememberSaveableStateHolderNavEntryDecorator()` — `rememberSaveable` inside entries silently stops persisting state → [Nav3 save state](https://developer.android.com/guide/navigation/navigation-3/save-state)
- Anonymous or non-top-level `NavKey` destination types — breaks process-death restore and `rememberSaveable` for that entry → [Nav3](https://developer.android.com/guide/navigation/navigation-3)
- Feature/screen ViewModel owns or mutates the back stack directly — navigation lifecycle coupled to ViewModel, defeats predictive back and scene strategies → [Nav3](https://developer.android.com/guide/navigation/navigation-3)
- `ResultEventBus` / `conflateAsState` result treated as surviving process death — result is only available within the current navigation lifetime; persist in `SavedStateHandle` if survival is required → [Nav3 save state](https://developer.android.com/guide/navigation/navigation-3/save-state)

Name Nav3 state findings as **Nav3 backstack state** in the State section.

Suggested interpretation:

- `9-10`: strong UDF, clear ownership, minimal ambiguity
- `7-8`: mostly healthy with some hoisting or saveability gaps
- `4-6`: repeated ownership confusion or weak state boundaries
- `0-3`: systemic duplication, stale data risks, or non-observable state misuse

### Side Effects

All citations in this category point to the canonical [Side-effects in Compose](https://developer.android.com/develop/ui/compose/side-effects) page unless noted. The official docs put `derivedStateOf` and `snapshotFlow` under side-effects. v1 keeps `derivedStateOf` weighted under Performance because that's where its value most often lands in audits, but the side-effects category also looks at it for *misuse*. Pick the dominant category and cross-reference rather than double-counting.

Reward:

- side-effect-free composition → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- correct use of `LaunchedEffect`, `DisposableEffect`, `SideEffect`, `rememberUpdatedState`, and `produceState` → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- effects keyed to the right lifecycle inputs → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- cleanup for listeners, observers, and subscriptions → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- stale callback capture avoided with `rememberUpdatedState` when needed → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- `snapshotFlow { … }` collected from inside a `LaunchedEffect` for Compose-state → Flow conversions → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- `rememberCoroutineScope()` used only for event-driven work (button taps, gesture handlers); long-lived/keyed work lives in `LaunchedEffect` → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- navigation, snackbar, analytics, and repository calls live in event handlers or `LaunchedEffect`, never in the composition body → [side-effects](https://developer.android.com/develop/ui/compose/side-effects), [navigation](https://developer.android.com/develop/ui/compose/navigation)
- image loading delegated to threading-aware loaders (Coil, Glide) via their Compose integration APIs (`AsyncImage`, `rememberAsyncImagePainter`) — these manage the threading contract correctly and are the accepted carve-out from "no IO in composition" → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- suspending UI / ViewModel paths preserve structured cancellation: broad catches and `runCatching` rethrow `CancellationException` before converting failures into UI state → [coroutines](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)
- `Animatable.animateTo(...)` / `snapTo(...)` driven from a `LaunchedEffect(key)` so the animation cancels on key change and on composition exit → [animation](https://developer.android.com/develop/ui/compose/animation/value-based), [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- declarative targets use `animate*AsState` and leave coroutine management to Compose; manual `rememberCoroutineScope().launch { ... }` is reserved for event-driven or gesture-driven animation → [animation](https://developer.android.com/develop/ui/compose/animation/value-based), [side-effects](https://developer.android.com/develop/ui/compose/side-effects)

Deduct for:

- launching threads, coroutines, navigation, or external work directly in composition → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- **strictly forbidden IO in a composable body** — never appears in composition regardless of how it is wrapped (`remember` does not redeem it, since the work still runs on the composition thread on first composition and on every key change). Categories: serialization/deserialization (`Json.decodeFromString`, `Gson`, `Moshi`, `ObjectMapper`, XML/Protobuf/CSV); network access (`HttpClient`, `OkHttp`, `Retrofit`, `URL(...)`, `Socket`); database connections or queries (`DriverManager`, `Connection`, `prepareStatement`, `executeQuery`, raw Room/SQLite calls); file or directory IO (`File(...)`, `Files.*`, `FileInputStream`, `BufferedReader`); subprocess / process IO (`ProcessBuilder`, `Runtime.exec`). Threading-aware image loaders (Coil, Glide) used via their Compose integration APIs are the explicit carve-out → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- wrong effect API choice → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- incorrect or missing effect keys → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- stale captures in long-lived effects → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- empty or suspicious `onDispose` → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- listeners or observers registered without cleanup → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- `snapshotFlow { … }` invoked outside an effect, or used to compute values that `derivedStateOf` would handle more cheaply → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- `rememberCoroutineScope()` used to launch work that should live in a keyed `LaunchedEffect` (manual cancellation, lifecycle handling reinvented) → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- `derivedStateOf { a + b }`-style misuse where inputs change as often as outputs — pure overhead per the official guidance → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- `LaunchedEffect(Unit)` / `LaunchedEffect(true)` flagged only when the body captures parameter or state values that may change without being keyed or wrapped in `rememberUpdatedState`. The "run once on enter" pattern itself is idiomatic; do not deduct for it → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- `rememberUpdatedState` read eagerly inside `remember { ... }` and assigned to a captured local before invocation — the remembered object keeps the initial callback/value forever. Read the `State.value` at invocation time or key the remembered object on the source value → [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- `runCatching { ... }`, `catch (Exception)`, or `catch (Throwable)` around suspend calls without rethrowing `CancellationException` — cancellation is converted into an error state instead of propagating through the coroutine hierarchy → [coroutines](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)
- `navController.navigate(...)` invoked from the composition body instead of an event handler or effect → [navigation](https://developer.android.com/develop/ui/compose/navigation)
- `rememberCoroutineScope().launch { animatable.animateTo(target) }` where a `LaunchedEffect(target) { animatable.animateTo(target) }` would more clearly express target-driven animation — prefer `LaunchedEffect(target)` when restart semantics should follow the key automatically; keep `rememberCoroutineScope()` for event- or gesture-driven animation → [animation](https://developer.android.com/develop/ui/compose/animation/value-based), [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- `Animatable.animateTo(...)` invoked from the composition body (not from inside an effect) — runs on every recomposition and never cancels → [animation](https://developer.android.com/develop/ui/compose/animation/value-based), [side-effects](https://developer.android.com/develop/ui/compose/side-effects)
- unconditional `lazyPagingItems.refresh()` / `retry()` from the composable body or bare `LaunchedEffect(Unit)` without a documented one-shot reason — initial load belongs to `PagingData`; refresh is user-driven → [paging compose](https://developer.android.com/topic/libraries/architecture/paging/v3-compose), [side-effects](https://developer.android.com/develop/ui/compose/side-effects)

When this paging rule affects the Side Effects score, name it in the Side Effects section as **Paging side-effect signals** rather than folding it into a generic effect note — mirroring **Paging list correctness** (Performance) and **Paging load-state handling** (State).

**Navigation 3 deductions (Side Effects)**

When Nav3 is detected (see playbook section 2b), also deduct for:
- `backStack.add` / `backStack.removeLastOrNull` called directly in the composition body instead of inside an event handler or `LaunchedEffect` — navigation fires on every recomposition → [Nav3](https://developer.android.com/guide/navigation/navigation-3)
- Tap handler missing `dropUnlessResumed { backStack.add(...) }` — a queued tap can navigate from a screen already in exit animation, duplicating the back-stack entry → [Nav3](https://developer.android.com/guide/navigation/navigation-3)

Note: "feature ViewModel owns/mutates the back stack" is a **State Management** finding (see above), not Side Effects — do not score it here.

Name Nav3 side-effect findings as **Nav3 navigation side effects** in the Side Effects section.

Suggested interpretation:

- `9-10`: deliberate, lifecycle-aware effect usage
- `7-8`: mostly correct with small effect-key or cleanup issues
- `4-6`: recurring misuse of effects or composition-time work
- `0-3`: side effects commonly happen in composition or cleanup is broadly unsafe

### Composable API Quality

This category is lighter-touch for app repositories. Focus on shared internal components, UI kits, and reusable building blocks. The two authoritative sources are the [Compose API guidelines](https://developer.android.com/develop/ui/compose/api-guidelines) and the deeper [Component API guidelines](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md).

Reward:

- reusable components expose `modifier: Modifier = Modifier` → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- `modifier` is the first optional parameter and applied once as the first link in the chain on the root-most UI node → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- parameter order is sensible: required, `modifier`, optional, trailing content lambda → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- defaults are meaningful and not hidden behind nullable sentinel behavior; defaults exposed through a `ComponentDefaults` object → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- explicit parameters preferred over component-specific `CompositionLocal` indirection. `CompositionLocal` is appropriate for *tree-scoped* data with sensible defaults (theme tokens like `LocalContentColor`, `LocalTextStyle`); not appropriate for component-specific configuration → [compositionlocal](https://developer.android.com/develop/ui/compose/compositionlocal), [api-guidelines](https://developer.android.com/develop/ui/compose/api-guidelines)
- components are focused and layered instead of multipurpose grab-bags → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- slot APIs (`content: @Composable RowScope.() -> Unit`) used for flexible composition; receiver scopes (`RowScope`, `ColumnScope`, `BoxScope`) applied where they guide layout → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- `Basic*` naming for unstyled / minimal variants alongside the opinionated public version (e.g. `BasicTextField` ↔ `TextField`) → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- distinct components for visual variants (`ContainedButton`, `OutlinedButton`, `TextButton`) instead of a single component with a `style` enum → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- `@Composable` functions are PascalCase and Unit-returning where they emit UI → [api-guidelines](https://developer.android.com/develop/ui/compose/api-guidelines)
- UI-emitting composables either emit UI or report via callbacks/state holders, but do not also return handles or computed values → [api-guidelines](https://developer.android.com/develop/ui/compose/api-guidelines)
- reusable content composables emit a cohesive root node, or explicitly declare a parent-layout scope when they intentionally emit multiple siblings → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- event callbacks are named and positioned as events (`onClick`, `onValueChange`) while trailing lambdas are reserved for composable content slots → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- custom modifiers built with `Modifier.Node` rather than the discouraged `composed { }` factory → [custom modifiers](https://developer.android.com/develop/ui/compose/custom-modifiers)
- `movableContentOf` / `movableContentWithReceiverOf` used to preserve slot-content lifecycle when content moves between containers → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- reusable APIs prefer `value: T` (immediate read) or `value: () -> T` (deferred read) plus `onValueChange: (T) -> Unit` over `MutableState<T>` parameters → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- isolated components define `@Preview` configurations to prove they render stateless → [tooling](https://developer.android.com/develop/ui/compose/tooling/previews)
- reusable animated components expose `animationSpec: AnimationSpec<T> = spring()` (or another appropriate default) when callers may reasonably need timing control, and use meaningful labels on shared/tooling-visible animations → [animation](https://developer.android.com/develop/ui/compose/animation/customize)

Deduct for:

- shared components missing `modifier` → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- multiple modifier params or modifier applied to the wrong child (anywhere other than the root-most emitted layout) → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- hardcoded strings in UI elements (e.g. `Text("Share with friends")` instead of `stringResource(id = R.string...)`) which break i18n support and create brittle tests → [resources](https://developer.android.com/develop/ui/compose/resources)
- hardcoded magic numbers like `.padding(12.dp)` or `.size(24.sp)` or explicit `Color(0xFF...)` instead of routing through `MaterialTheme` tokens or `dimensionResource`. A clean theme is vital for dark mode and accessibility → [theming](https://developer.android.com/develop/ui/compose/designsystems/material3)
- no `@Preview` coverage for extracted UI chunks. If a component is reusable, it should have a preview proving it has no hidden ambient dependencies → [tooling](https://developer.android.com/develop/ui/compose/tooling/previews)
- `modifier` with a non-no-op default like `Modifier.padding(8.dp)` (caller's modifier silently loses the padding) → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- parameter ordering that makes APIs awkward or misleading → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- nullable params used only as "use internal default" signals — expose the default explicitly via a `ComponentDefaults` object instead → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- passing raw network models, database entities, or complex domain objects directly into UI components instead of mapping them to stable, UI-specific presentation models (UiState). This leaks backend structure into the presentation tier, encourages giant data models, and often forces the Compose compiler to treat the arguments as unstable. → [architecture](https://developer.android.com/develop/ui/compose/architecture)
- `MutableState<T>` or `State<T>` params in reusable APIs when avoidable; the official replacement is `value: T` or `value: () -> T` plus `onValueChange: (T) -> Unit` → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- giant multipurpose components that should be split or wrapped → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- UI-emitting composables that return non-`Unit` values or handles. Report state via callbacks / state holders; keep value-returning composables value-only → [api-guidelines](https://developer.android.com/develop/ui/compose/api-guidelines)
- reusable content composables that emit multiple sibling roots without an explicit parent scope such as `RowScope` or `ColumnScope`; wrap in a cohesive root layout or make the scope contract explicit → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- pure helpers marked `@Composable` even though they neither read composition data nor emit UI, or `@ReadOnlyComposable` applied to functions that emit UI / launch effects / mutate state → [api-guidelines](https://developer.android.com/develop/ui/compose/api-guidelines)
- event callbacks placed as the trailing lambda so call sites read like content slots, or callbacks named as past-tense notifications when Compose-style `onXxx` event names would make the contract clearer → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- behavior added as parameters that should be modifiers (`onClick`, `clipToCircle`) → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- style/configuration objects passed to a single component instead of distinct components per variant → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- non-Compose lifecycles attached to composables — for example, an `onClick` callback on a layout component when `Modifier.clickable` would do → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- modifier ordering that quietly changes semantics in shared components (e.g. `Modifier.padding(...).clickable {}` extends the click region into the padding; `Modifier.clickable {}.padding(...)` does not). Flag when the choice looks accidental → [custom modifiers](https://developer.android.com/develop/ui/compose/custom-modifiers)
- hardcoded `dp`, `sp`, or explicit `Color` constructs in reusable components instead of using `MaterialTheme.colorScheme`, `MaterialTheme.typography`, or dimension resources; this destroys dark mode compliance and accessible font scaling → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- `CompositionLocal` used for component-specific configuration (vs. truly tree-scoped data); ViewModels stored in `CompositionLocal`; locals with no sensible default → [compositionlocal](https://developer.android.com/develop/ui/compose/compositionlocal)
- custom modifiers built with `Modifier.composed { }` when `Modifier.Node` would do — `composed { }` is officially discouraged for performance → [custom modifiers](https://developer.android.com/develop/ui/compose/custom-modifiers)
- `Scaffold { innerPadding -> ... }` content that does not apply `innerPadding` to its root child (or consume it via `consumeWindowInsets`) — content draws behind the `TopAppBar` / `BottomAppBar` / system bars → [component API](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md)
- highly reusable animated composables that hard-code `tween(N)` / specific `spring()` stiffness in the signature when callers would reasonably need control over timing — consider exposing `animationSpec: AnimationSpec<T>` instead → [animation](https://developer.android.com/develop/ui/compose/animation/customize)
- shared/tooling-visible `animate*AsState` / `updateTransition` call sites missing a meaningful `label` parameter — reduces clarity in Animation Preview and other Android Studio tooling; treat as a light deduction, not a hard failure → [animation](https://developer.android.com/develop/ui/compose/animation/customize)

Suggested interpretation:

- `9-10`: shared UI APIs are clean, predictable, and reusable
- `7-8`: solid conventions with a few inconsistencies
- `4-6`: repeated API smell in shared components
- `0-3`: shared component APIs are confusing, rigid, or actively error-prone

## Scoring Rules

- One isolated issue should not tank a whole category.
- Repeated issues across several modules should count more than a single bad file.
- Production code matters more than samples, previews, or scratch files.
- Positive patterns should raise confidence and may offset minor issues.
- App teams may intentionally deviate from framework/library guidance. Note the tradeoff before deducting heavily.
- **Every deduction in the report must include a `References:` line citing one or more URLs from the rubric above (or from `references/canonical-sources.md`).** A deduction without a citation should not appear in the report.
