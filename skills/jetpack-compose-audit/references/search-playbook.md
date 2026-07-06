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
- Compose UI tests, screenshot tests, focus/keyboard tests, and semantics test helpers
- KMP / Compose Multiplatform source sets (`commonMain`, `androidMain`, `iosMain`, `desktopMain`, `wasmJsMain`) and platform interop leaves

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
- `collectAsLazyPagingItems|LazyPagingItems`

If Compose usage is sparse, state that clearly in the report and reduce confidence.

## 2a. Map Adjacent Coverage Notes

These areas are not scored in v1, but the audit should not be blind to them. Record presence, obvious gaps, and focused follow-ups in `Notes And Limits` / `Suggested Follow-Up`.

### Testing Surface

Search for:

- `createComposeRule|createAndroidComposeRule`
- `onNodeWithText|onNodeWithTag|onAllNodes|assertIsDisplayed|assertDoesNotExist|performClick|performTextInput`
- `assertIsFocused|performKeyInput|pressKey`
- `Paparazzi|Roborazzi|Screenshot|golden`
- `@Preview|PreviewLightDark|PreviewFontScale|PreviewScreenSizes`

Notes to add:

- If reusable components or screen UI exist with no nearby UI tests/previews, note the gap but do not deduct.
- If UI tests construct a full app graph for simple rendering/callback checks, recommend `compose-agent focus on testing`.
- If screenshots rely on real network, clocks, random data, or live image loading, note likely flakiness and recommend deterministic fakes.

### Focus / Keyboard Surface

Search for:

- `FocusRequester|focusRequester|focusProperties|focusable|onFocusChanged`
- `onPreviewKeyEvent|onKeyEvent|Key\.Direction|Key\.Back|Key\.Escape`
- `assertIsFocused|performKeyInput|pressKey`
- `androidx\.tv|TvLazy|Dpad|Remote|ChromeOS|desktop`

Notes to add:

- If focus APIs exist without key-input tests, recommend `compose-agent focus on focus`.
- If `requestFocus()` appears outside an effect, this can be a Side Effects deduction because it performs imperative work during composition.
- If lazy-list focus appears index-based in reorderable content, note a focus-restoration risk.

### KMP / Compose Multiplatform Surface

Search for:

- `commonMain|androidMain|iosMain|desktopMain|wasmJsMain`
- `expect\s+|actual\s+`
- `AndroidView|UIKitView|ComposeUIViewController`
- `android\.content\.Context|Activity|UIViewController|NSBundle|Uri|Bundle` in common source sets
- `collectAsStateWithLifecycle` inside common source sets

Notes to add:

- Android-only APIs in `commonMain` are a KMP boundary risk; recommend `compose-agent focus on kmp`.
- Platform interop should stay at leaf composables with lifecycle handled by effects.
- Do not deduct for Android-only APIs in Android-only modules. Only note the constraint when the repo is shared/KMP.

### Android Launch UX / Splash Surface

This is adjacent coverage, not one of the four score categories. Still run it during normal audits because launch resources are outside Kotlin and easy to miss.

Search XML resources for:

- `windowSplashScreenAnimatedIcon|android:windowSplashScreenAnimatedIcon`
- `windowSplashScreenBackground|postSplashScreenTheme`
- `<animated-vector|<vector|<adaptive-icon|<bitmap|<layer-list`

Detection heuristic:

1. Find theme/style files with the splash icon item:
   `rg -n 'windowSplashScreenAnimatedIcon' -g 'themes.xml' -g 'styles.xml' -g '*.xml'`
2. Resolve each `@drawable/name` reference by checking resource qualifiers in this order for Android 12+: `res/drawable-v31/name.xml`, then other API 31+ drawable qualifiers, then unqualified `res/drawable/name.xml` / density drawables.
3. Read the resolved API 31+ drawable XML root. Pass when it is `<animated-vector>`.
4. Flag when the API 31+ resource resolves to a static `<vector>`, `<adaptive-icon>`, bitmap, or `layer-list`, or when the unqualified static drawable is used with no `drawable-v31` animated-vector override.

Report format:

- Title: **Android Launch UX: Android 12+ static splash icon may render blurry**
- Evidence: theme file line, referenced resource, resolved drawable file, and missing/present `drawable-v31` override.
- Fix direction: keep the same `@drawable/name` in the theme and make it resolve to an `<animated-vector>` on API 31+ that wraps the real vector. **No animators are required — an empty `<animated-vector>` is enough**, because the platform draws any `Animatable` icon at full size instead of via the `ImmobileIconDrawable` 108 dp pre-render. The wrapper's `android:drawable` must point at a **differently-named** vector (referencing its own theme name self-resolves on API 31+ and loops). This is using the attribute as the docs recommend (an `AnimatedVectorDrawable`), not just a hack.
- References: `https://developer.android.com/develop/ui/views/launch/splash-screen`, `https://developer.android.com/reference/androidx/core/splashscreen/SplashScreen`, and the platform issue `https://issuetracker.google.com/issues/520672537`.

Do not subtract from Performance, State, Side Effects, or Composable API Quality. It may still belong in `Critical Findings` or `Prioritized Fixes` because it is user-visible and usually cheap to fix.

## 2b. Navigation 3 Detection & Audit

Run this section **only when Nav3 is present**. Nav3 ships a different model from Nav2 — many wrong patterns compile fine but break backstack restore, process death recovery, or recomposition safety at runtime. (Note: string routes and typed-key mismatches are often compile errors in Nav3; the runtime risks are primarily state/save-state related.)

### Detect Nav3 Usage

```bash
rg -l 'rememberNavBackStack|NavDisplay|entryProvider|NavKey|androidx\.navigation3' -g '*.kt' -g '*.kts'
```

If any file matches, Nav3 is in scope. Continue below. If none match, skip this section.

### Map Nav3 Entry Points

```bash
# Back stack instantiation
rg 'rememberNavBackStack' -g '*.kt' -n

# NavDisplay host(s)
rg 'NavDisplay\s*\(' -g '*.kt' -n

# Destination key types (multiline: @Serializable often on separate line from : NavKey)
rg -B1 ': NavKey' -g '*.kt' -n

# Custom entry decorators
rg 'entryDecorators\s*=' -g '*.kt' -n

# Result bus usage
rg 'rememberResultEventBusNavEntryDecorator|LocalResultEventBus|ResultEffect|conflateAsState' -g '*.kt' -n
```

### Red Flags To Verify

**Backstack ownership violation — feature ViewModel holds or mutates the back stack**

```bash
rg 'NavBackStack|backStack' -g '*.kt' -l | xargs -r rg -l 'ViewModel|viewModel\(\)' 2>/dev/null
```

A feature/screen ViewModel must never own or directly mutate the back stack. Navigation signals should flow from the ViewModel as state/events that the *route* observes and acts on via `backStack.add` / `removeLastOrNull`. A dedicated app-level nav holder that *owns* the stack is a legitimate pattern; a feature ViewModel that receives `backStack` as a parameter for navigation is not.

**High false-positive rate:** any file with both `backStack` and `viewModel()` matches — including `App.kt` where the nav holder and ViewModel co-exist legitimately. Always read the hit to confirm the ViewModel actually holds or mutates the stack, not merely that both symbols appear in the same file.

Severity: **Blocker** — this couples navigation lifecycle to the ViewModel and defeats predictive back and scene strategies.

---

**`entryDecorators` supplied without re-adding `rememberSaveableStateHolderNavEntryDecorator`**

```bash
rg 'entryDecorators\s*=' -g '*.kt' -n -A 6
```

For each hit: check whether the decorator list includes `rememberSaveableStateHolderNavEntryDecorator()`. Supplying a custom list *replaces all defaults*, so `rememberSaveable` inside entries silently stops working if the default saveable-state decorator is omitted.

Severity: **Blocker** — `rememberSaveable` inside affected entries persists nothing; crash or data loss on config change.

---

**Navigation triggered in composition body (not from event handler or `LaunchedEffect`)**

```bash
rg 'backStack\.(add|removeLastOrNull|clear)\b' -g '*.kt' -n
```

For each hit: confirm it is inside a lambda passed as an event handler (e.g. `onClick`, `onBack`, `onOpenProfile`) or inside a `LaunchedEffect`. A call directly in the composition body triggers navigation on every recomposition.

Severity: **Blocker** — navigation fires on every recomposition, duplicates back-stack entries.

---

**Missing `dropUnlessResumed` click guard**

```bash
# -U enables multiline matching so backStack.add can be on a different line than onClick
rg -U 'onClick\s*=\s*\{[^}]*backStack\.add' -g '*.kt' -n
rg -U 'on\w+\s*=\s*\{[^}]*backStack\.add' -g '*.kt' -n
# Simpler alternative: find all backStack.add call sites and read surrounding context
rg 'backStack\.add\(' -g '*.kt' -n -B3
```

For each navigation-triggering tap handler: check it is wrapped in `dropUnlessResumed { backStack.add(...) }`. Regex matches are hints — multiline lambdas and nested braces require reading the actual code around each hit. Without this guard, a queued tap can navigate from a screen that has already left `RESUMED`, pushing a duplicate entry during the exit animation.

Severity: **Should-fix** — rare in practice but reproducible with fast double-taps and on slow devices.

---

**Anonymous or non-top-level destination keys**

```bash
# Catch anonymous objects (object expression, not object declaration)
rg 'object\s*:\s*NavKey' -g '*.kt' -n
# Catch NavKey usages that are not top-level data class/object or sealed interface/class base types
rg ': NavKey' -g '*.kt' -n | grep -Ev '(data (class|object)|sealed (class|interface)|^.*object\s+\w+\s*:)'
```

All `NavKey` **destination** types must be top-level `@Serializable` data classes or objects. Anonymous inline keys (object expressions) and local classes break `rememberSaveable` and cannot be restored after process death.

Valid patterns that must **not** be flagged: `sealed interface AppNavKey : NavKey`, `object HomeKey : NavKey` (named top-level object), base sealed hierarchies. Read each hit before filing — sealed/interface base types are always valid.

Severity: **Blocker** for anonymous objects and local classes — process-death restore fails; `rememberSaveable` inside that entry silently stops working.

---

**Nav2 `NavController` / `NavHost` used alongside Nav3 code**

```bash
rg 'rememberNavController|NavHost\s*\(' -g '*.kt' -n
```

Check whether any hit coexists in the same navigation graph with `NavDisplay`. Mixing Nav2 and Nav3 in the same back-stack flow is not supported; they must own separate, non-overlapping regions of the UI.

Severity: **Blocker** if mixed in the same flow; **Nit** if clearly isolated sub-graphs.

---

**String routes still used in Nav3 code**

```bash
# Scope to Nav3 files only (from the detection query above) to avoid flagging Nav2 sub-graphs
rg 'backStack\.add\s*\(\s*"' -g '*.kt' -n
# navigate("…") is a Nav2 API; flag only if found in a file that also uses NavDisplay/rememberNavBackStack
rg 'navigate\s*\(\s*"' -g '*.kt' -l | xargs -r rg -l 'NavDisplay|rememberNavBackStack' 2>/dev/null
```

Nav3 destinations are `@Serializable` typed keys — not strings. `backStack.add("…")` won't compile with `NavBackStack<NavKey>`, so a hit almost always indicates dead/migrated code or a type mismatch. `navigate("…")` in a file that also uses `NavDisplay` signals an incomplete Nav2→Nav3 migration.

Severity: **Should-fix** (migration smell / likely dead code) rather than Blocker — Nav3's type system makes the string-route pattern a compile error in practice.

---

**`@Composable` or non-serializable type inside a NavKey destination**

```bash
# Find NavKey destination types first
rg -l ': NavKey' -g '*.kt' | xargs -r rg -n '@Composable|\(\s*\)\s*->\s*\w+|\([^)]+\)\s*->\s*\w+|suspend\s*\(' 2>/dev/null
```

Destination data classes must be serializable plain data. Composable references, lambdas (`() -> Unit`, `(String) -> Unit`, suspend lambdas), or any non-serializable type as a field will crash at serialization time. Scope the search to NavKey files to avoid noise from non-navigation data classes.

Severity: **Blocker** — runtime crash on process death or `rememberSaveable`.

---

**`ResultEventBus` result assumed to survive process death**

```bash
rg 'conflateAsState|ResultEffect' -g '*.kt' -n
```

`conflateAsState` delivers the latest result within the current navigation lifetime — it is **not** persisted across process death. If a result must survive process death, it belongs in a persisted state holder (e.g. `SavedStateHandle`), not just in `conflateAsState`.

Severity: **Should-fix** when the result represents user data that must not be lost on backgrounding.

### Positive Signals

- All destinations are top-level `@Serializable data class` / `object` implementing `NavKey`
- `dropUnlessResumed { backStack.add(...) }` on every tap-driven navigation call
- `rememberSaveableStateHolderNavEntryDecorator()` present in every custom `entryDecorators` list
- App-level nav holder owns the back stack; feature ViewModels expose navigation state as events/state, not by holding `backStack`
- `rememberViewModelStoreNavEntryDecorator()` added when per-entry `viewModel()` scoping is needed (and placed after the saveable-state decorator)
- `ResultEventBus` used for screen-to-screen results; outcomes that must survive process death persisted in `SavedStateHandle`

### Nav3 Scoring Note

Nav3 findings map to existing score categories — do not open a new one:

| Finding | Category |
|---------|----------|
| Feature ViewModel owns/mutates back stack | **State Management** (ownership of state) |
| `entryDecorators` missing saveable-state decorator | **State Management** |
| Anonymous / non-top-level `NavKey` types | **State Management** |
| `ResultEventBus` result assumed to survive process death | **State Management** |
| Navigation triggered in composition body | **Side Effects** |
| Missing `dropUnlessResumed` click guard | **Side Effects** (Should-fix) |

Report State findings as **Nav3 backstack state**; report Side Effects findings as **Nav3 navigation side effects**.

Do not open a new score category for Nav3. If the project has no Nav2 code at all, note in the report that Nav2 navigation patterns are out of scope and Nav3 is the only navigation layer audited.

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
- cross-phase back-writes — a **layout- or draw-phase write into state an earlier composition read**, re-invalidating it: a layout callback (`onSizeChanged`, `onGloballyPositioned`, `onPlaced`) writes a `mutableStateOf` / `mutableIntStateOf` that a **sibling reads in composition** → measure → write → recompose loop, often per-frame during resize/scroll. This is axis 3 proper (a later phase feeding an earlier one). See the heuristic below
- composition-phase self-invalidation — a **`SnapshotStateMap` / `SnapshotStateList` mutated inside a `@Composable` body** (`putAll`, `put`, `add`, `addAll`, `remove`, `clear`, `[k] =`, `+=`, `-=`) that the same composition also reads. This is *not* cross-phase — the write and read are both in composition; it is a close cousin of the same-body backwards-write rule above. Score it **once**: if the body reads the collection and *then* mutates it, the existing backwards-write rule already covers it — flag it here only when the mutation-then-read / self-feeding shape would slip past that rule's "already read" wording
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

### Paging-List Heuristic

When `collectAsLazyPagingItems` / `LazyPagingItems` is present, paging-specific lazy-list bugs are in scope even if generic lazy-list searches were clean.

1. Find paging UI files: `rg -l 'collectAsLazyPagingItems|LazyPagingItems' -g '*.kt'`
2. For each hit, verify:
   - `items(` / `itemsIndexed(` uses **`key =`** with stable domain ids (`itemKey { … }`), not index-only keys
   - UI branches on **`loadState`** (refresh / append / error) — not a blank list during initial load or after failure
   - **`refresh()`** / **`retry()`** are not invoked unconditionally from the composable body or a bare `LaunchedEffect(Unit)` without a documented one-shot reason
   - no **`indexOf` / `indexOfFirst`** inside the paging item factory
   - the append branch handles **`LoadState.Error`** with a `retry()` footer, not only `LoadState.Loading`
   - a full-screen `refresh is LoadState.Error` branch is **guarded by `itemCount == 0`** — otherwise a transient pull-to-refresh failure wipes an already-loaded feed
   - no **`lazyPagingItems[index]!!`** — null slots occur when `PagingConfig.enablePlaceholders = true`; check the null handling matches the config
   - no **`itemSnapshotList`** wired as the primary list source — it materializes the differ and defeats paging; treat as a list-correctness smell
3. When keys are missing or index-based on a paginated feed, or `itemSnapshotList` drives the primary UI, flag under Performance as **Paging list correctness** and recommend `compose-agent focus on paging`.
4. When `LoadState.Error` is ignored or loading never resolves, flag under State management as **Paging load-state handling**.
5. When `refresh()` / `retry()` is invoked from composition or a bare `LaunchedEffect(Unit)`, flag under Side Effects as **Paging side-effect signals** (initial load is `PagingData`'s job; these are user-initiated).

Positive signals to reward:

- `items(count = lazyPagingItems.itemCount, key = lazyPagingItems.itemKey { it.id })`
- top-level `when` on `loadState.refresh` / `loadState.append` with error retry via `lazyPagingItems.retry()`
- pull-to-refresh wired to `lazyPagingItems.refresh()` on user action only

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
   - the same composable hosts multiple `animate*AsState` calls all driven by the same boolean/enum — check whether `updateTransition` would keep them synchronized.

Positive signals to reward:

- `updateTransition(targetState, label = "...")` with `transition.animateFloat` / `transition.animateColor` when several properties move together

- `Modifier.graphicsLayer { alpha = alphaAnim.value; rotationZ = rot.value }` for alpha / rotation / scale animations
- `Modifier.offset { IntOffset(offset.value.roundToInt(), 0) }` for position animations
- `remember { Animatable(...) }` + `LaunchedEffect(target) { animatable.animateTo(target) }` pattern
- `AnimatedContent(targetState, label = "...")` when the call site needs custom enter/exit or size-aware transitions; `Crossfade` for standard fade-only swaps
- reusable animated components exposing `animationSpec: AnimationSpec<T>` when callers need timing control, and using meaningful labels on tooling-visible animations

### Cross-Phase Back-Write & Composition Self-Invalidation Heuristic

Two related re-invalidation smells. Only shape 1 is truly cross-phase (axis 3); shape 2 stays within composition and overlaps the same-body backwards-write rule — keep them distinct so the report neither mislabels nor double-deducts. No single regex proves either; gather candidates, then read each call site.

1. **Layout → composition writes (cross-phase, axis 3).** Layout runs after composition, so a layout-phase write into state read in composition re-runs composition. List the layout callbacks: `rg -n 'onSizeChanged|onGloballyPositioned|onPlaced' -g '*.kt'`. For each hit, check whether its lambda **writes** a state holder (`something = ...`, `state.value = ...`, `.intValue =`) rather than only reading, then confirm that written state is **read during composition** — by a sibling composable, a parent, or the same subtree (not only inside a deferred lambda modifier). It is legitimate only when the value is read exclusively in a later layout/draw pass (e.g. fed straight into `Modifier.layout { }` / `Modifier.drawBehind { }`); flag it when a composition-phase read exists.
2. **Snapshot-collection self-invalidation (composition phase, *not* cross-phase).** First find snapshot collections: `rg -n 'mutableStateListOf|mutableStateMapOf|SnapshotStateList|SnapshotStateMap' -g '*.kt'`. Then, **only inside the files that hit above** (a global mutation search is pure noise — every `list.add(...)` in the repo matches), read each usage and flag a mutation — `putAll` / `put` / `add` / `addAll` / `remove` / `clear` / `[k] =` / `+=` / `-=` — that sits **directly in a `@Composable` body** (not in an event handler, `LaunchedEffect`, or state holder) while the same composition also reads the collection. Deduct **once**: prefer the same-body backwards-write rule when the body reads before it mutates; use this shape for the mutate-then-read / self-feeding case that the "already read" wording would miss.

Severity: **Should-fix**, promoted to a **Critical Finding** when the loop is per-frame (resize/scroll/animation-driven) or spans reused lazy items. For shape 1, cite the writer's `file:line` and the reader's `file:line`. Docs: [phases](https://developer.android.com/develop/ui/compose/performance/phases), [bestpractices](https://developer.android.com/develop/ui/compose/performance/bestpractices).

Positive signals to reward:

- a measured size / position written from a layout callback is consumed only in `Modifier.layout { }` / `Modifier.offset { }` / `Modifier.drawBehind { }` (stays in layout/draw, never re-enters composition)
- `SubcomposeLayout` / `BoxWithConstraints` used where a child genuinely needs parent constraints, instead of round-tripping measured size through composition-read state

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
- `runCatching\s*\{`
- `catch\s*\(\s*\w+\s*:\s*(Throwable|Exception)\s*\)`
- `CancellationException`
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
- `LaunchedEffect(Unit)` / `LaunchedEffect(true)` is **not** suspicious on its own (the "run once" pattern is idiomatic). Only flag it when the body captures parameter or state values that may change without `rememberUpdatedState` — **except** `LaunchedEffect(Unit) { lazyPagingItems.refresh() / retry() }`, which is a Side Effects defect (paged initial load is `PagingData`'s job; `refresh()`/`retry()` are user-initiated)
- `DisposableEffect` with empty or suspicious cleanup
- listener/observer registration without `onDispose`
- effect keys too broad or too narrow
- `rememberCoroutineScope()` used to launch keyed/long-lived work that belongs in a `LaunchedEffect`
- `rememberUpdatedState` read eagerly inside `remember { ... }` (for example, assigning `val captured = latestCallback` inside the remembered block) — this captures the initial callback instead of reading the latest value at invocation time
- `runCatching { ... }`, `catch (Exception)`, or `catch (Throwable)` around suspend calls in UI-reached code without rethrowing `CancellationException` — cancellation becomes an error state and structured concurrency breaks
- event handlers in leaf composables directly calling repositories, persistence, network clients, or other durable business services — route through a ViewModel / state holder callback unless the caller explicitly owns the suspending boundary
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
- non-Unit composable signatures: `@Composable[\s\S]{0,400}?fun\s+\w+\s*\([\s\S]*?\)\s*:\s*@?\w` — note `[^)]*` would stop at the first `)`, missing every composable whose params contain a lambda type (`onClick: () -> Unit`); the non-greedy `[\s\S]*?` spans the whole param list. Verify hits by eye, since this can also span past the intended signature.
- composition marker annotations: `@ReadOnlyComposable`, `@NonRestartableComposable`
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
- UI-emitting composables that also return a value or handle — expose callbacks / state holders instead
- reusable content composables that emit multiple sibling roots without an explicit parent-layout scope such as `RowScope` / `ColumnScope`
- pure helper functions marked `@Composable` even though they do not read composition data, snapshot state, or emit UI
- `@ReadOnlyComposable` on any function that emits UI, launches effects, or mutates state
- event callbacks placed as the trailing lambda where the call site looks like a content slot; keep trailing lambdas for `@Composable` content
- `movableContentOf` used for changing caller content without a freshness strategy such as `rememberUpdatedState(content)` read inside the movable block

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
