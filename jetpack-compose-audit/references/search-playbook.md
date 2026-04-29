# Search Playbook

Use this playbook to gather evidence quickly, then verify by reading representative files.

**Always scope ripgrep to Kotlin sources** with `-g '*.kt' -g '*.kts'` (or the `Grep` tool's `glob` parameter / `type: kotlin`) — the regexes below are written for Kotlin and produce noise on JVM/Java/JS files. For build-config searches, use `-g '*.gradle*' -g '*.toml'` instead.

## 1. Map The Repo First

Before category scoring, locate:

- Gradle settings and module declarations
- Android app and feature modules
- likely Compose source folders
- shared component or design-system directories
- theme packages
- screen packages
- ViewModels and state holders
- test and preview directories

Useful targets often include:

- `app/`
- `feature*/`
- `core/ui/`
- `designsystem/`
- `ui/components/`
- `ui/screens/`
- `presentation/`

## 2. Confirm Compose Surface Area

Search for:

- `@Composable`
- `MaterialTheme`
- `setContent`
- `ComposeView`
- `remember`
- `mutableStateOf`

If Compose usage is sparse, state that clearly in the report and reduce confidence.

## 3. Performance Checks

### Search For

- `remember\(`
- `derivedStateOf`
- `LazyColumn|LazyRow|LazyVerticalGrid|LazyHorizontalGrid|LazyVerticalStaggeredGrid|LazyHorizontalStaggeredGrid`
- `items\(`
- `itemsIndexed\(`
- `rememberLazyListState`
- `animate`
- `offset\(`
- `drawBehind`
- `sortedWith|sortedBy|filter|map|associate|groupBy`
- stability annotations and immutable collections: `@Stable`, `@Immutable`, `kotlinx\.collections\.immutable`, `ImmutableList`, `PersistentList`, `persistentListOf`, `compose_compiler_config`
- skipping opt-outs: `@NonSkippableComposable`, `@DontMemoize`
- typed state factories: `mutableIntStateOf`, `mutableLongStateOf`, `mutableFloatStateOf`, `mutableDoubleStateOf`
- composition-marker annotations: `@ReadOnlyComposable`, `@NonRestartableComposable`
- baseline / profile setup: `baselineProfile`, `ProfileInstaller`, `androidx\.profileinstaller`, `baseline-prof\.txt`
- deprecated/legacy APIs: `accompanist-pager`, `accompanist-swiperefresh`, `accompanist-flowlayout`, `accompanist-systemuicontroller`, `animateItemPlacement\(`
- config-derived reads inside `remember {}`: `LocalConfiguration`, `LocalDensity`, `LocalLayoutDirection`
- `\.indexOf\(|\.lastIndexOf\(|\.indexOfFirst\s*\{` inside lazy item factories
- O(N) string work in composable bodies: `\.split\(`, `\.lines\(`, `\.lineSequence\(`, `\.trimIndent\(`, `\.replace\(`, `\.format\(`, `\.substringBefore`, `\.substringAfter` — flag any hit inside a `@Composable` body; trivially cheap reads (`\.size`, `\.length`, `\.isEmpty\(\)`, index access) are fine
- inline regex compilation/execution: `Regex\(`, `\.toRegex\(`, `\.matches\(`, `\.find\(` — flag when the construction or call sits inside a composable body. Hoist the compiled `Regex` or move the matching upstream
- O(N) collection aggregates in composable bodies: `\.count\s*\{`, `\.joinToString\(`, `\.mapIndexed\(`, `\.flatMap\(`, `\.sumOf\(`, `\.maxOf\(`, `\.minOf\(`, `\.fold\(`, `\.reduce\(` — flag in `@Composable` bodies, especially over collections whose size is unbounded
- `Canvas\s*\(` and `Spacer\s*\(` — check each hit for an explicit `size` / `height` / `aspectRatio` on the modifier; bare `fillMaxSize()` on a drawing surface can enter draw with `Size.Zero`
- animation APIs: `animate\w+AsState\b`, `updateTransition\b`, `rememberInfiniteTransition\b`, `Animatable\b`, `AnimatedVisibility\b`, `AnimatedContent\b`, `Crossfade\b`
- animation-to-modifier hand-off: lines where `\.value\b` from an animated state (`animate\w+AsState`, `Animatable`, `updateTransition`) is passed to `Modifier.offset\(`, `Modifier.alpha\(`, `Modifier.rotate\(`, `Modifier.scale\(`, `Modifier.padding\(` — verify whether the lambda-form (`Modifier.offset { ... }`, `Modifier.graphicsLayer { ... }`) would defer the read to the layout/draw phase
- `ReportDrawnWhen\s*\{` — positive signal for startup metrics
- `enableEdgeToEdge\s*\(` — positive signal; also confirms the project is not reaching for the deprecated `accompanist-systemuicontroller`

### Red Flags To Verify

- expensive list transforms inside composable bodies
- expensive work passed directly to `items(...)`
- `Lazy*` items without `key =` where item identity can move or reorder — see the lazy-list-without-key heuristic at the bottom of this section
- scroll or animation state read high in the tree
- fast-changing values passed to non-lambda modifiers when a layout/draw-phase alternative exists
- backwards writes — *writing to state that has already been read in the same composition body* (this is the precise definition; reading after writing is fine)
- `mutableStateOf<Int>` / `<Long>` / `<Float>` / `<Double>` — the typed factories avoid boxing
- raw `List`/`Map`/`Set` parameters on widely reused composables when `kotlinx.collections.immutable` is already a dependency
- `@NonSkippableComposable` / `@DontMemoize` without a justifying comment
- `remember { … }` whose body reads `LocalConfiguration` / `LocalDensity` / `LocalLayoutDirection` without listing that source as a key — cached value goes stale on rotation/foldable/font-scale changes
- `indexOf(...)` / `lastIndexOf(...)` / `indexOfFirst { ... }` called inside a `LazyListScope` item factory — O(n²) scrolling cost and crash risk if identity moves; prefer `itemsIndexed`
- `animateItemPlacement()` usage on Compose 1.7+ — replaced by `Modifier.animateItem()`
- Accompanist libraries where first-party replacements exist: `accompanist-pager` → `HorizontalPager` / `VerticalPager`; `accompanist-swiperefresh` → `PullToRefreshBox`; `accompanist-flowlayout` → `FlowRow` / `FlowColumn`; `accompanist-systemuicontroller` → `enableEdgeToEdge()`
- `Animatable\s*\(` created in a composable body without `remember { ... }` or hoisting — each recomposition creates a fresh `Animatable` and restarts the animation
- animated `.value` read in composition body and piped to state-reading modifiers: `Modifier\.(offset|alpha|rotate|scale|padding)\s*\(\s*\w+\.value` — prefer lambda-form modifiers to defer per-frame reads to layout/draw
- `rememberInfiniteTransition\s*\(` hosted outside of an `AnimatedVisibility` / lazy-list item / `if (visible)` guard — verify that the host actually leaves composition when offscreen; if not, the animation keeps running until removal and does needless offscreen work
- `Crossfade\s*\(` used where the call site appears to need custom enter/exit or size-aware transitions — consider whether `AnimatedContent` is the better API

### Positive Signals

- `remember(keys)` around expensive calculations
- `derivedStateOf` used for scroll-triggered UI thresholds
- lambda modifiers such as `Modifier.offset { ... }`, `Modifier.graphicsLayer { ... }`, `Modifier.drawBehind { ... }`
- draw/layout phase reads instead of full recomposition for rapidly changing values
- `@Stable` / `@Immutable` on data classes used as composable params
- `ImmutableList` / `PersistentList` for collection params
- `compose_compiler_config.conf` to mark third-party types stable
- baseline-profile modules or profile installer setup when app maturity suggests it matters

### Lazy-List-Without-Key Heuristic

There's no clean single regex for this. Use a two-step approach:

1. Find files that use a lazy layout: `rg -l 'Lazy(Column|Row|VerticalGrid|HorizontalGrid|VerticalStaggeredGrid|HorizontalStaggeredGrid)\b' -g '*.kt'`
2. Within each file, look for `items(` / `itemsIndexed(` invocations that omit `key =`. Useful multiline pattern: `rg -U --multiline-dotall 'items(?:Indexed)?\([^)]*\)' -g '*.kt'` and read each hit for a `key =` argument.

Manually verify before deducting — `items(count: Int)` overloads and small static lists that never reorder are not bugs.

### Duplicate-Lazy-Key Heuristic

Compose throws `IllegalArgumentException: Key ... was already used` when a `Lazy*` layout sees two items with the same key. Root causes in production code: backend returning duplicate IDs, merging streams (e.g. WebSocket reconnect), or `Pager` + `LazyColumn` combinations where the same item appears in overlapping pages.

There is no clean regex. Walk `items(..., key = ...)` / `itemsIndexed(..., key = ...)` hits and read the surrounding context:

- is the list source a merge / combine / concatenation of multiple flows?
- does the backend spec guarantee ID uniqueness?
- is the key computed from `hashCode()` on a non-`data class`?

When uniqueness is not guaranteed, flag as a latent crash and suggest a dedup index or a synthesized key like `"${source}-${id}"`.

### Scaffold Inner-Padding Heuristic

`Scaffold` exposes `innerPadding` to its content lambda. If the content ignores it, elements are drawn behind the `TopAppBar` or `BottomAppBar`. Search for `Scaffold(` and read each hit — the content lambda parameter should be applied to the root-most child via `Modifier.padding(innerPadding)` (or `.consumeWindowInsets(innerPadding)`). If a `Scaffold { }` discards the padding parameter with `_ ->` or omits it entirely while nesting non-trivial content, flag it.

### Animation Phase-Smell Heuristic

Per-frame animated values belong in the layout or draw phase, not in recomposition. No clean single regex — walk the animation call sites and read each one:

1. List files with animation APIs: `rg -l 'animate\w+AsState|updateTransition|Animatable|rememberInfiniteTransition' -g '*.kt'`
2. For each hit, locate where the animated `.value` is consumed. Flag when:
   - `val x by animate*AsState(...)` feeds `Modifier.offset(x.dp)`, `Modifier.alpha(x)`, `Modifier.rotate(x)`, `Modifier.scale(x)`, `Modifier.padding(x.dp)` — the state read happens at composition time, so every frame recomposes the caller. Lambda-form alternatives (`Modifier.offset { IntOffset(...) }`, `Modifier.graphicsLayer { ... }`) defer the read to layout / draw.
   - `Animatable(...)` is created in a composable body without `remember { ... }` or hoisting.
   - `animateTo(...)` / `snapTo(...)` is launched from the composition body rather than a `LaunchedEffect` or an event handler.
   - `rememberInfiniteTransition()` is hosted in a composable that stays composed offscreen (tab host, drawer content that is cheaper to keep mounted) — the animation keeps running until the host is removed from composition, with no visible benefit while offscreen.

Positive signals to reward:

- `Modifier.graphicsLayer { alpha = alphaAnim.value; rotationZ = rot.value }` for alpha / rotation / scale animations
- `Modifier.offset { IntOffset(offset.value.roundToInt(), 0) }` for position animations
- `remember { Animatable(...) }` + `LaunchedEffect(target) { animatable.animateTo(target) }` pattern
- `AnimatedContent(targetState, label = "...")` when the call site needs custom enter/exit or size-aware transitions; `Crossfade` for standard fade-only swaps
- reusable animated components exposing `animationSpec: AnimationSpec<T>` when callers need timing control, and using meaningful labels on tooling-visible animations

### Strong Skipping Mode Check

Confirm the project's compiler version:

- Find the Kotlin version: `rg -n 'kotlin\s*=\s*"' -g '*.toml'` and `rg -n 'org\.jetbrains\.kotlin' -g '*.gradle*'`.
- Find any explicit Strong Skipping toggle: `rg -n 'enableStrongSkippingMode|StrongSkipping' -g '*.gradle*' -g '*.properties'`.
- Strong Skipping is on by default at Kotlin **2.0.20+** / Compose Compiler **1.5.4+**. Earlier versions can opt in via `-Pandroidx.compose.compiler.enableStrongSkippingMode=true` or the equivalent `freeCompilerArgs` flag.

Scoring implications — decide once, apply consistently through the report:

- **Strong Skipping OFF (pre-2.0.20 or explicitly disabled).** Stability inference matters; unstable params aggressively block skipping. Deduct normally for raw `List` / `Map` / `Set` params on reused composables, unstable `data class` params, and missing `@Stable` / `@Immutable` annotations.
- **Strong Skipping ON.** All restartable composables become skippable and lambdas are auto-memoized via `==` / instance equality. Wrapping `List` in `@Stable` classes, using `ImmutableList`, or wrapping lambdas in `remember { ... }` is no longer required *just to enable skipping*. **Do not** deduct for raw `List` parameters or missing `@Stable` annotations under SSM on their own.
- **Under Strong Skipping, deduct instead for the two real hot-path smells:**
  - **Instance-recreation churn.** `listOf(...)`, `mapOf(...)`, `setOf(...)`, fresh object literals, or `MyParams(...)` allocated inside a composable body and passed as a param. Search with `rg -n 'listOf\(|mapOf\(|setOf\(' -g '*.kt'` inside files that also contain `@Composable`, and read each hit — if the call sits in a composable body (not in a `remember { }`, state holder, or ViewModel) and the result is forwarded to a child composable, flag it.
  - **Expensive or broken `equals()` on unstable params.** Plain `class Foo(...)` (identity equality) passed to reusable composables, `data class` wrapping a large collection (deep `==` per recomposition), or `data class` with `var` / mutable fields (stale skip results). Cross-reference the compiler report's unstable-classes list against each unstable class's definition to triage.

## 4. State Management Checks

### Search For

- `mutableStateOf`
- `mutableIntStateOf|mutableLongStateOf|mutableFloatStateOf|mutableDoubleStateOf`
- `mutableStateListOf|mutableStateMapOf`
- `rememberSaveable`
- `mapSaver|listSaver|@Parcelize|Saver`
- `collectAsStateWithLifecycle`
- `collectAsState`
- `observeAsState`
- `subscribeAsState`
- `MutableState<`
- `State<`
- `mutableListOf|mutableMapOf|mutableSetOf|ArrayList`
- `CompositionLocal`
- `compositionLocalOf|staticCompositionLocalOf`
- `ViewModel`
- `viewModel\(` — log invocation depth (screen entry vs. deep tree)
- `mutableStateOf` / `mutableIntStateOf` etc. declared as members of a class extending `ViewModel` (not inside a composable)
- `Channel<` / `SharedFlow<` / `receiveAsFlow\(\)` / `consumeAsFlow\(\)` exposed from a `ViewModel` for UI events — verify whether the outcome should instead be durable UI state
- `\.stateIn\s*\(` — positive signal; check for `WhileSubscribed(5_000)` or similar timeout
- `rememberSaveable` invoked inside a `Lazy(Column|Row|VerticalGrid|HorizontalGrid|VerticalStaggeredGrid|HorizontalStaggeredGrid)` item factory

### Red Flags To Verify

- duplicated state across parent/child or sibling composables
- shared state held too low in the tree
- reusable components with unnecessary internal state
- Android UI code using `collectAsState()` where `collectAsStateWithLifecycle()` is a better fit (skip this rule on Compose Multiplatform code paths)
- non-observable mutable collections used as state (`mutableListOf` mutated in place)
- `mutableListOf` / `mutableMapOf` wrapped in a `MutableState` instead of `mutableStateListOf` / `mutableStateMapOf` — element changes won't trigger recomposition
- `mutableStateOf<Int|Long|Float|Double>` instead of the typed factory (autoboxing)
- `MutableState<T>` params in reusable components
- `State<T>` params where a value or lambda would be more flexible
- `remember { ... }` with no `key` for a value that depends on inputs (stale cache)
- `rememberSaveable { mutableStateOf(SomeNonBundleable(...)) }` without a `Saver` — restoration silently fails

### Positive Signals

- clear stateful wrapper + stateless reusable composable split
- state hoisted to the lowest common reader / highest writer
- related state hoisted together
- lifecycle-aware collection of flows in Android UI
- plain state-holder classes for larger screens or app shells

### Flow-Misuse-In-UI Heuristic

Flow operators belong in presenters / state holders / ViewModels, not composable bodies. Pipelines constructed in a `@Composable` are rebuilt on every composition unless wrapped in `remember(...)` with correct keys — and even when they're correctly remembered, the data layer is the wrong place for shaping logic. Two questions, separate answers: *where do you collect?* (the UI edge, via `collectAsStateWithLifecycle()` or a keyed `LaunchedEffect`) and *where do you shape?* (`combine`, `flatMapLatest`, `stateIn`, `debounce`, etc. — in a presenter / ViewModel, never inline in a composable). A composable with three or more Flow operator imports is doing presenter work.

Two-step audit:

1. List files that mix Compose with Flow construction:
   `rg -l '@Composable' -g '*.kt' | xargs -r rg -l 'combine\(|merge\(|zip\(|flatMapLatest|flatMapMerge|flatMapConcat|stateIn\(|shareIn\(|debounce\(|sample\(|buffer\(|conflate\(|collectLatest|MutableStateFlow|MutableSharedFlow|Channel\b'`
2. For each hit, decide whether the operator is inside a composable body or a class member / top-level helper. Flag a deduction when:
   - shaping operator chains (`combine`, `merge`, `flatMapLatest`, `flatMapMerge`, `debounce`, `sample`, `buffer`, `conflate`) are constructed inside an `@Composable` body — move to the presenter
   - terminal collection (`collectLatest`, `collect`) happens directly in a composable body; collection inside a correctly keyed `LaunchedEffect(...)` for ephemeral events is acceptable
   - `stateIn(...)` / `shareIn(...)` are started from a composition-scoped scope (`rememberCoroutineScope()`, inside `LaunchedEffect`) — pipeline is torn down on disposal and restarted on next entry
   - `MutableStateFlow` / `MutableSharedFlow` / `Channel` are constructed inside a composable rather than owned by a state holder / ViewModel
   - `MutableStateFlow` / `MutableSharedFlow` / `Channel` are exposed publicly from a ViewModel without `asStateFlow()` / `asSharedFlow()` / `receiveAsFlow()` narrowing
   - `MutableSharedFlow<UiEvent>` (replay-0) is used for outcomes that must not be lost across STOPPED — payment results, deletion confirmations, save success. Convert to durable state with an `onConsumed` callback.
   - `zip(...)` is used to build screen state, or `combine(...)` is used with cold inputs that may never emit — combined flow gets stuck on `initialValue`
   - `flatMapMerge` is used where `flatMapLatest` should cancel the previous in-flight work (search-as-you-type, route changes, user-id swaps)

Positive signals to reward:

- a single `StateFlow<UiState>` per screen, exposed via `private val _state = MutableStateFlow(...); val state = _state.asStateFlow()` and constructed via `combine(...).stateIn(viewModelScope, WhileSubscribed(5_000), initial)` inside a presenter / ViewModel
- composables reading that state with `viewModel.state.collectAsStateWithLifecycle()` and otherwise free of operator chains
- `Channel<Event>` private, exposed as `Flow<Event>` via `receiveAsFlow()`, used only for genuinely ephemeral UI commands

## 5. Side Effects Checks

### Search For

- `LaunchedEffect`
- `DisposableEffect`
- `SideEffect`
- `rememberUpdatedState`
- `produceState`
- `snapshotFlow`
- `rememberCoroutineScope`
- `\.launch\s*[\({]` — catches both `scope.launch {` and `scope.launch(Dispatchers.IO) {`
- `Thread\(`
- `GlobalScope`
- `LifecycleEventObserver`
- `BackHandler`
- `NavHost`, `composable\(` (in nav graphs), `navController\.navigate`
- string-based nav routes: `composable\(\s*"` and `navigate\(\s*"` (suggest type-safe `@Serializable` routes on Navigation Compose 2.8+)
- `\.animateTo\s*\(` / `\.snapTo\s*\(` on `Animatable` instances — check each hit is inside a `LaunchedEffect(...)` (correct) or a gesture / event handler (acceptable), not the composition body
- `rememberCoroutineScope\(\)` followed nearby by `\.launch\s*\{[^}]*animateTo` — likely a target-driven animation written imperatively; check whether `LaunchedEffect(target)` would express restart semantics more clearly
- IO classes that should never appear in a composable body — flag any hit inside a `@Composable` function (Coil/Glide via their Compose integration APIs are the accepted carve-out):
  - serialization: `Json\.`, `Gson`, `Moshi`, `ObjectMapper`, `Xml`, `Protobuf`, `\bCsv\b`
  - network: `HttpClient`, `OkHttp`, `Retrofit`, `URL\(`, `Socket\(`
  - filesystem: `File\(`, `Files\.`, `FileInputStream`, `BufferedReader`, `Paths\.get`
  - database: `DriverManager`, `\bConnection\b`, `prepareStatement`, `executeQuery`
  - subprocess: `ProcessBuilder`, `Runtime\.getRuntime`

### Red Flags To Verify

- work started directly in composition body
- **strictly forbidden IO in a composable body** — serialization, network, database, file/directory IO, or subprocess work. `remember` is **not** a valid fix: the cached body still runs on the composition thread on first composition and on every key change. Threading-aware image loaders (Coil's `AsyncImage` / `rememberAsyncImagePainter`, Glide's Compose API) are the explicit carve-out
- navigation, snackbar, analytics, or repository calls triggered during composition instead of from an effect or event path
- `navController.navigate(...)` invoked in composition body — must come from an event handler or `LaunchedEffect`
- `LaunchedEffect(Unit)` / `LaunchedEffect(true)` is **not** suspicious on its own (the "run once" pattern is idiomatic). Only flag it when the body captures parameter or state values that may change without `rememberUpdatedState`
- `DisposableEffect` with empty or suspicious cleanup
- listener/observer registration without `onDispose`
- effect keys too broad or too narrow
- `rememberCoroutineScope()` used to launch keyed/long-lived work that belongs in a `LaunchedEffect`
- `snapshotFlow { ... }` invoked outside an effect, or used to compute a value that `derivedStateOf` would handle more cheaply
- `derivedStateOf { a + b }`-style misuse — when input frequency ≈ output frequency it is pure overhead (the official antipattern)
- an `Animatable` animation launched from the composition body (not inside `LaunchedEffect` or an event handler) — it will restart with recomposition and bypass effect lifecycle semantics
- `rememberCoroutineScope().launch { animatable.animateTo(...) }` where the animation reacts purely to a `targetState` change — `LaunchedEffect(targetState) { animatable.animateTo(targetState) }` is often the clearer fit because restart semantics follow `targetState` automatically

### Positive Signals

- `rememberUpdatedState` used for long-lived effects that should keep latest callbacks
- `DisposableEffect` paired with clear cleanup
- `SideEffect` used only to publish state to non-Compose code after successful composition
- `produceState` or equivalent used for converting external async sources into Compose state
- `snapshotFlow { ... }` collected from inside a `LaunchedEffect` for Compose-state → Flow conversions
- `rememberCoroutineScope()` used only for event-driven work (button taps, gesture handlers)

## 6. Composable API Quality Checks

Focus on shared components and internal UI kit code, not every screen.

### Search For

- shared component directories such as `components`, `commonui`, `designsystem`, `ui/components`
- function signatures around `@Composable`
- `modifier: Modifier =`
- `Modifier = Modifier\.` — non-no-op modifier defaults
- `MutableState<`
- `State<`
- `CompositionLocal`
- `compositionLocalOf|staticCompositionLocalOf` — definitions
- `CompositionLocalProvider` — provision sites
- ViewModels in CompositionLocal: `compositionLocalOf<.*ViewModel`, `staticCompositionLocalOf<.*ViewModel`
- `viewModel\(` — invocation sites; flag when called below the screen entry composable
- slot APIs and receiver scopes: `RowScope\.`, `ColumnScope\.`, `BoxScope\.`, `content:\s*@Composable`
- modifier authoring: `Modifier\.composed\s*\{` (discouraged), `Modifier\.Node`, `ModifierNodeElement`
- movable content: `movableContentOf`, `movableContentWithReceiverOf`
- variant smells: `\bstyle:\s*\w+Style\b` — single-component-with-style-enum
- `Basic` prefix: `fun Basic[A-Z]\w+\s*\(`
- lazy list `contentType`: `contentType\s*=` inside `items(` / `itemsIndexed(` calls — its presence on heterogeneous lists is a positive signal; its absence on heterogeneous lists is a deduction

### Red Flags To Verify

- shared components missing a `modifier`
- `modifier` not being the first optional parameter
- `modifier` default not equal to `Modifier` (e.g. `modifier: Modifier = Modifier.padding(8.dp)` — caller's modifier silently loses the padding)
- multiple modifier parameters on one component
- modifier applied to a child instead of the root-most emitted UI, and not as the *first* link in the chain
- nullable params used to mean "choose internal default" (expose a `ComponentDefaults` object instead)
- component-specific configuration hidden behind implicit locals
- huge multipurpose components that should be layered or split
- behavior added as parameters that should be modifiers (`onClick`, `clipToCircle`)
- a single component with a `style: ButtonStyle` enum-like parameter instead of distinct `ContainedButton` / `OutlinedButton` / `TextButton` components
- custom modifiers built with `Modifier.composed { ... }` (discouraged in favor of `Modifier.Node`)
- `MutableState<T>` params — replace with `value: T` (immediate read) or `value: () -> T` (deferred) plus `onValueChange: (T) -> Unit`
- `@Composable` UI-emitting functions named in lowerCamelCase or returning a non-Unit value (style guide violation)

### Positive Signals

- required params first, then `modifier`, then optional params, then trailing content lambda
- explicit config exposed as parameters with meaningful defaults via a `ComponentDefaults` object
- focused component responsibilities
- internal wrappers built on simpler lower-level components
- slot lambdas (`content: @Composable RowScope.() -> Unit`) for flexible composition
- `Basic*` variants alongside opinionated public versions
- distinct components per visual variant (no `style` enum)
- `movableContentOf` used to preserve slot lifecycle across structural moves
- custom modifiers authored with `Modifier.Node` / `ModifierNodeElement`

### Modifier-Order Smell

No clean regex. When reading shared components, watch for `Modifier.padding(...).clickable {}` vs `Modifier.clickable {}.padding(...)` — they produce different ripple bounds and hit areas. Flag when the choice looks accidental in a reusable component (the wrong order is almost always a bug there; in a one-off screen it may be intentional).

## 7. Read Representative Files

For each category, read enough code to cover:

- at least one screen
- at least one shared component area
- at least one state-owning area such as a ViewModel or state-holder
- any suspicious files surfaced by search

Do not rely on one "bad" file to characterize the entire repo.

## 8. Merge Repeated Findings

Prefer:

- "7 shared components miss `modifier`"

over:

- seven separate bullets saying the same thing

Still keep 2-4 concrete file examples to justify the systemic finding.

## 9. Out-Of-Scope Cases

Stop or narrow scope if:

- the repo is not Android Jetpack Compose
- Compose is only present in demo or sample code
- the user asked to audit only one module or feature

In those cases, explain the limitation and score only the relevant surface area.
