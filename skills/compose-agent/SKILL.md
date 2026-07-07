---
name: compose-agent
description: "Helps AI coding assistants write modern Jetpack Compose: correct state, side effects, performance-aware modifiers, Navigation 3, Paging 3 in Compose, coroutines on lifecycle, animations, UI tests, focus/keyboard navigation, Compose Multiplatform boundaries, idiomatic Kotlin, and well-shaped composable APIs. Targets the mistakes LLMs actually make in Compose code. Use when reading, writing, or reviewing Compose projects."
license: MIT
allowed-tools: Read, Glob, Grep, Edit, Write, Bash
argument-hint: "[focus area, e.g. 'state', 'effects', 'navigation', 'paging', 'lifecycle', 'animation', 'testing', 'focus', 'kmp']"
metadata:
  author: Ivan Morgillo
  version: "4.3.2"
---

# Compose Agent

Review, write, or modify Jetpack Compose code for correctness, modern API usage, and adherence to the official AndroidX guidelines. Report only genuine problems — do not nitpick or invent issues.

**Target track (unless the repo says otherwise):**

- Kotlin `2.0.20+` with Compose Compiler `1.5.4+` (Strong Skipping Mode on by default).
- Jetpack Compose BOM current, Material 3, `androidx.lifecycle` with `collectAsStateWithLifecycle()`.
- Navigation 3 (`androidx.navigation3`) when the project can adopt it; otherwise Navigation 2.8+.
- Coroutines + Flow as the async primitives. Do not reach for RxJava or blocking I/O.

If the repo pins older versions, match the repo — but call out what the modern path would look like in a one-line note.

## Review Process

1. Check for **deprecated or soft-deprecated API** using `references/api.md`.
2. Validate **state and data flow** using `references/state.md`.
3. Validate **side-effect choice** (`LaunchedEffect`, `DisposableEffect`, `produceState`, `snapshotFlow`, `rememberUpdatedState`) using `references/effects.md`.
4. Review **composable performance** — stability, lambda modifiers, lazy list keys, deferred reads, cross-phase back-writes — using `references/performance.md`.
5. Review **modifier usage** — ordering, lambda-form, `Modifier.Node` over `composed { }` — using `references/modifiers.md`.
6. Review **navigation** using `references/navigation.md` — start from its decision table (Nav3 vs Nav2 type-safe vs plain state) before touching call sites.
7. Review **coroutines and lifecycle collection** using `references/concurrency.md`.
8. Review **Flow operators and StateFlow / SharedFlow shape** (`stateIn`, `shareIn`, `flatMap` variants, `combine`, error handling, backpressure, `asStateFlow()`) using `references/flows.md`.
9. Review **composable API shape** (parameter order, slots, naming, defaults) using `references/component-api.md`.
10. Review **UI testing patterns** using `references/testing.md` when tests, semantics, screenshots, previews, or fake UI dependencies are in scope.
11. Review **focus and keyboard / D-pad navigation** using `references/focus.md` when the UI uses focus APIs, keyboard input, TV, desktop, ChromeOS, or accessibility focus behavior.
12. Review **Compose Multiplatform / KMP boundaries** using `references/kmp.md` when common code, `expect` / `actual`, platform services, native views, or shared UI targets are in scope.
13. Review **animation API choice and lifecycle** — declarative vs imperative, `remember`ed `Animatable`, target-driven launches, `spring` over `tween`, deferred animated reads, `AnimatedContent` keys — using `references/animation.md` when any `animate*`, `Animatable`, `Transition`, `AnimatedVisibility`, `AnimatedContent`, or infinite-transition API is in scope.
14. Review **Paging 3 in Compose** — `collectAsLazyPagingItems` (the paging differ — **not** `collectAsStateWithLifecycle`, which does not apply to `PagingData`), stable `itemKey`, `LoadState` branches, user-driven `refresh()` / `retry()` — using `references/paging.md` when `LazyPagingItems`, `PagingData`, or `collectAsLazyPagingItems` is in scope.
15. Final **Kotlin style** pass using `references/kotlin.md`.

If doing a partial review, load only the relevant reference files — each references file is designed to be read in isolation.

## Core Instructions

- Keep composables **side-effect free** in composition. All I/O, state mutation outside remembered objects, analytics, etc. belong in `LaunchedEffect`, `DisposableEffect`, `produceState`, the ViewModel, or — for **event/gesture-driven** work only — `rememberCoroutineScope().launch` inside a handler. Target-driven animation belongs in `LaunchedEffect` or declarative `animate*AsState` / `updateTransition`, not in `scope.launch` from the composition body. See `references/animation.md` and `references/effects.md`.
- Every composable that takes a `Modifier` names the parameter `modifier`, types it `Modifier = Modifier`, and applies it **to the outermost layout only**. Never create a `modifier` parameter you do not forward.
- Parameter order for a component: required data → `modifier: Modifier = Modifier` → other optional parameters with defaults → trailing `content: @Composable () -> Unit` slots last. One `modifier` parameter per component.
- For stateful APIs, expose a stateless overload plus a stateful convenience that hoists `remember`. See `references/component-api.md`.
- Collect Flows with `collectAsStateWithLifecycle()` in UI code. Plain `collectAsState()` keeps collecting when the screen is not visible and burns battery and bandwidth. **Exception:** `Flow<PagingData<T>>` is collected with `collectAsLazyPagingItems()`, never `collectAsStateWithLifecycle()` — see `references/paging.md`.
- Prefer `rememberSaveable` over `remember` for UI state that should survive configuration change or process death, unless the value is unserializable or derivable.
- Use the typed state factories (`mutableIntStateOf`, `mutableLongStateOf`, `mutableFloatStateOf`, `mutableDoubleStateOf`) for primitive state. Raw `mutableStateOf<Int>(...)` boxes.
- Never write state a phase has already read. A layout callback (`onSizeChanged` / `onGloballyPositioned` / `onPlaced`) must not write state a sibling reads in composition (a cross-phase back-write, a measure → recompose loop), and never mutate a `mutableStateListOf` / `mutableStateMapOf` inside a composable body that also reads it (composition-phase self-invalidation, a mutate → recompose loop). Consume measured values in layout/draw, or mutate snapshot state from an event / `LaunchedEffect` / state holder. See `references/performance.md`.
- Don't ship no-op "optimizations": `remember(index)` on a pure function, or `remember`-ing an already-auto-memoized callback under Strong Skipping (SSM memoizes lambdas even with unstable captures). Prove a recomposition win with Layout Inspector counts or compiler reports, not the presence of a pattern. See `references/performance.md`.
- Never put a lambda in a `CompositionLocal`. Use explicit parameters.
- Do not introduce third-party libraries without asking first. The Accompanist libraries covering pager, swipe-refresh, flow layout, and system UI controller are **deprecated** — the functionality is in AndroidX now (`HorizontalPager`, `PullToRefreshBox`, `FlowRow`/`FlowColumn`, `enableEdgeToEdge()`).
- When an element's background reads from `MaterialTheme.colorScheme.*` (directly or via a blend), its text and icon colors must read from the same theming source. Hard-coded `Color.Black` / `Color.White` / raw ARGB literals over theme-driven backgrounds are dark-mode regressions. See `references/component-api.md`.
- Animate declaratively first. Use `animate*AsState` for single state-driven values and `updateTransition` for synchronized ones; reach for `Animatable` only when you need imperative control (gesture, fling, sequence). Always `remember` an `Animatable`, launch target-driven animations from `LaunchedEffect` (never `scope.launch` in composition to react to state), prefer `spring()` over `tween` for interruptible motion, and read animated values in the layout/draw phase (lambda modifiers / `graphicsLayer`) so the animation does not recompose every frame. See `references/animation.md`.
- For paged feeds, use `collectAsLazyPagingItems()`, render with `items(count = …, key = …)` and stable domain ids, branch on `loadState` for loading/empty/error/append, and keep `refresh()` / `retry()` on user actions — not in the composition body. See `references/paging.md`.

## Output Format (review mode)

Organize findings by file. For each issue:

1. State the file and relevant line(s).
2. Name the rule being violated (e.g., "Use `collectAsStateWithLifecycle()` instead of `collectAsState()`").
3. Show a brief before/after code fix.
4. Link the official source (`developer.android.com/...` or AndroidX component guidelines).

Skip files with no issues. End with a prioritized summary — **three** items max, highest impact first.

### Example Output

#### `feature/profile/ProfileScreen.kt`

**Line 34: Use `collectAsStateWithLifecycle()` — UI Flows must stop collecting when the screen is not STARTED.**

```kotlin
// Before
val user by viewModel.user.collectAsState()

// After
val user by viewModel.user.collectAsStateWithLifecycle()
```

<https://developer.android.com/topic/architecture/ui-layer/state-production#continuous-vs-discrete>

**Line 58: Pass `modifier` to the outermost layout, not to inner content.**

```kotlin
// Before
@Composable
fun UserCard(user: User, modifier: Modifier = Modifier) {
    Column {
        Text(user.name, modifier = modifier) // modifier swallowed here
        Text(user.email)
    }
}

// After
@Composable
fun UserCard(user: User, modifier: Modifier = Modifier) {
    Column(modifier = modifier) {
        Text(user.name)
        Text(user.email)
    }
}
```

<https://developer.android.com/develop/ui/compose/api-guidelines#naming-modifiers>

**Line 72: Defer the animated `offset` read to the layout phase using the lambda form.**

```kotlin
// Before
val dx by animateDpAsState(targetValue = if (expanded) 0.dp else (-200).dp, label = "drawerOffset")
Box(modifier = Modifier.offset(x = dx))

// After — layout-phase read, skips recomposition on every frame
val dx by animateDpAsState(targetValue = if (expanded) 0.dp else (-200).dp, label = "drawerOffset")
Box(modifier = Modifier.offset { IntOffset(dx.roundToPx(), 0) })
```

<https://developer.android.com/develop/ui/compose/performance/bestpractices#defer-reads-as-long-as-possible>

### Summary

1. **Performance (high):** Non-deferred animated reads on `ProfileScreen.kt:72`, `HomeScreen.kt:110`, `DetailScreen.kt:88` recompose every frame. Switch to lambda-form modifiers.
2. **Lifecycle (high):** Three screens use `collectAsState()` — replace with `collectAsStateWithLifecycle()` to avoid collecting in the background.
3. **API shape (medium):** `UserCard` swallows its `modifier` parameter. Forward it to the outermost layout.

End of example.

## Authoring Mode

When the agent is **writing new code** rather than reviewing, the same rules apply as guardrails. Before generating a composable, silently check:

- Does it take `modifier: Modifier = Modifier`?
- Is state hoisted, or is there a clear reason to own it here?
- If it renders a list, does it use a stable `key =`?
- If it launches work, is that work in a `LaunchedEffect`, `produceState`, or the ViewModel — not in the composition body?
- If it collects a Flow, is it `collectAsStateWithLifecycle()`? (A `Flow<PagingData<T>>` is the exception — it uses `collectAsLazyPagingItems()`.)
- Is the parameter order: data → modifier → other → content slot last?
- If it animates, is the API declarative first, remembered where needed, lifecycle-aware, and phase-correct?
- Does any layout callback or snapshot-collection mutation write state read back in composition (a cross-phase / self-invalidation loop)?
- If it pages, are keys stable, `LoadState` handled, and refresh/retry user-driven?
- If it needs focus, testing, or platform-specific behavior, did you load the focused reference before judging?

If any answer is no and there is no deliberate reason, fix it before returning the code.

## What This Skill Does Not Cover

Out of scope in v1 — delegate elsewhere or scope down explicitly:

- **Material 3 compliance, theming, and design tokens** — use the `material-3` skill.
- **Scoring an existing codebase with numeric grades** — use the sibling `jetpack-compose-audit` skill in the same repo.
- **Wear OS / TV / Auto / Glance deep platform review** — this skill now covers focus and keyboard/D-pad basics, but not full platform certification.
- **Accessibility deep review** — we flag obvious gaps (missing `contentDescription`, icon-only buttons without labels, touch targets under 48dp) but do not grade.

If the user needs any of the above, narrow the scope and say so.

## References

- `references/api.md` — deprecated and soft-deprecated Compose APIs and their modern replacements.
- `references/state.md` — hoisting, `remember` vs `rememberSaveable`, ViewModel boundaries, `derivedStateOf`.
- `references/effects.md` — `LaunchedEffect`, `DisposableEffect`, `SideEffect`, `produceState`, `snapshotFlow`, `rememberUpdatedState`.
- `references/performance.md` — stability, Strong Skipping Mode, lambda modifiers, lazy keys, typed state factories, deferred reads.
- `references/modifiers.md` — modifier ordering, lambda form, `Modifier.Node` over `composed { }`.
- `references/navigation.md` — decision table (Nav3 vs Nav2 type-safe vs plain state), Navigation 3, and what to replace from Navigation 2.
- `references/concurrency.md` — coroutines, Flow, `collectAsStateWithLifecycle`, `repeatOnLifecycle`, scope choice.
- `references/flows.md` — operator selection: `StateFlow` vs `SharedFlow` vs cold `Flow`, `stateIn(WhileSubscribed)`, `shareIn`, `flatMap` variants, `combine`/`merge`/`zip`, error handling, backpressure, `asStateFlow()` exposure.
- `references/component-api.md` — composable API guidelines: parameter order, slots, naming, defaults, state hoisting shape.
- `references/testing.md` — Compose UI tests, semantics assertions, screenshot tests, fake image/platform services, previews.
- `references/focus.md` — `FocusRequester`, focus restoration, keyboard / D-pad input, key handlers, and focus tests.
- `references/kmp.md` — Kotlin Multiplatform and Compose Multiplatform boundaries, `expect` / `actual`, interfaces, platform leaf composables.
- `references/animation.md` — animation API selection (`animate*AsState` vs `updateTransition` vs `Animatable`), `remember`ed animation state, target-driven launches, gesture-driven `snapTo`/`animateDecay`, `spring`/`tween`/`keyframes`, reduced motion, deferred animated reads, `AnimatedContent`/`AnimatedVisibility`, `animateItem()`, off-screen infinite transitions.
- `references/paging.md` — Paging 3 in Compose: when to page vs plain lists, `collectAsLazyPagingItems`, stable `itemKey`, `LoadState` UI, pull-to-refresh, anti-patterns for paginated lazy lists.
- `references/kotlin.md` — Kotlin coding conventions and Android Kotlin style the LLM keeps missing.

## Primary Sources

Every rule in this skill traces back to one of:

- `https://developer.android.com/develop/ui/compose` and its subpages
- `https://developer.android.com/kotlin/style-guide`
- `https://kotlinlang.org/docs/coding-conventions.html`
- `https://www.jetbrains.com/help/kotlin-multiplatform-dev/`
- `https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-api-guidelines.md`
- `https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md`

When a supplemental source (blog post, conference talk) disagrees with these, the primary sources win.
