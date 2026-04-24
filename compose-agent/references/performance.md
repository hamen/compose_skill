# Performance

Compose performance is not "avoid recomposition" — some recomposition is expected. It is "keep recomposition cheap, skippable, and off the per-frame hot path". The rules below are what to enforce when writing or reviewing code.

For a full numeric audit of an existing codebase, use the sibling `jetpack-compose-audit` skill. This file is the authoring guardrail.

## Strong Skipping Mode — The Baseline

On Kotlin `2.0.20+` with Compose Compiler `1.5.4+`, **Strong Skipping Mode is on by default**. Under SSM:

- Any restartable composable whose parameters have all been structurally equal to the previous call is skipped.
- Raw `List<T>` / `Map<K, V>` / `Set<T>` parameters are no longer a hard blocker for skipping.
- `@Stable` / `@Immutable` annotations are no longer required on your own types to earn skippability.

What still defeats skipping under SSM:

1. **Fresh collection / fresh object literals in the call site.** `listOf(...)`, `mapOf(...)`, `MyUiModel(...)` built at the call site recompute a new identity on every recomposition. Structural equality may save you, but construction churn still costs.
2. **Object literals and lambda literals inside composable bodies.** Lambdas that capture state read inside the composable are automatically remembered by the compiler. Lambdas that capture nothing are singletons. Lambdas in between — capturing a value — are worth double-checking; the compiler memoizes most of them, but opaque call sites (e.g. generics) can still churn.
3. **Broken `equals()` on parameters.** If a data class overrides `equals()` incorrectly or is a plain class without `equals`, skipping fails for the wrong reason.
4. **Explicit `@NonSkippableComposable` / `@DontMemoize`** on hot paths.

If the repo has SSM **off** (older compiler or explicit opt-out), raw `List` / `Map` / `Set` parameters, missing `@Stable` on your types, and stateful shared collections all matter — follow the rules in the `jetpack-compose-audit` skill's `scoring.md`.

**To tell which mode applies,** check the Kotlin version and Compose Compiler version in the module's Gradle config. Strong Skipping can be toggled per module; assume on unless you see an explicit `freeCompilerArgs += listOf("-P", "plugin:androidx.compose.compiler.plugins.kotlin:strongSkipping=false")` or the legacy `enableStrongSkippingMode = false` flag.

## Hoist State Low, Read Lower

A recomposition cost is proportional to the **subtree that reads the state**. To keep it cheap:

- Read state as close to the UI that uses it as possible. If only a `Text` reads the counter, do not read the counter one level up and pass its value down.
- When the same state is read by two siblings, hoist to the nearest common ancestor — not higher.
- For values that travel through many layers, prefer passing `() -> T` (a state reader lambda) rather than `T`. The intermediate composables skip when the lambda identity is stable.

```kotlin
// Wrong — intermediate Row reads and recomposes on every count change
@Composable
fun Parent() {
    var count by remember { mutableIntStateOf(0) }
    MiddleRow(count = count)
}
@Composable fun MiddleRow(count: Int) { Row { Child(count) } }

// Right — MiddleRow only recomposes when its own structure changes
@Composable
fun Parent() {
    var count by remember { mutableIntStateOf(0) }
    MiddleRow(countProvider = { count })
}
@Composable fun MiddleRow(countProvider: () -> Int) { Row { Child(countProvider) } }
@Composable fun Child(countProvider: () -> Int) { Text("$countProvider()") }
```

## Defer Reads As Long As Possible

Every phase transition (composition → layout → draw) is an opportunity to skip work. Reading state later in the pipeline means earlier phases stay stable.

Lambda-form modifiers defer the read to layout or draw:

```kotlin
// offset
Modifier.offset(x = xDp)            // composition-phase read → recomposes every animation frame
Modifier.offset { IntOffset(x, 0) } // layout-phase read → composition is stable

// graphicsLayer — the generic escape hatch for alpha/scale/rotation/translation
Modifier.alpha(alpha)                  // composition
Modifier.graphicsLayer { this.alpha = alpha } // draw-phase read
```

Same idea for `padding`, `rotate`, `scale`. If the value is changing on every frame (animation, scroll), use the lambda form.

## Lazy Lists Need Keys

```kotlin
// Wrong — reorderable list without key: every item recomposes + animation breaks
LazyColumn {
    items(todos) { todo -> TodoRow(todo) }
}

// Right
LazyColumn {
    items(items = todos, key = { it.id }) { todo -> TodoRow(todo) }
}
```

Rules:

- Keys **must be stable and unique**. IDs from the domain are ideal. Do not use `hashCode()`, do not use the index.
- For heterogeneous lists (mixed item types), pass `contentType = { ... }` too — Compose reuses item layouts by content type.
- `key =` is also what makes `animateItem()` work. Missing keys → no item animations.

## Typed Primitive State

```kotlin
// Wrong — boxes Int into Integer
var count by remember { mutableStateOf(0) }

// Right
var count by remember { mutableIntStateOf(0) }
```

Same for `mutableLongStateOf`, `mutableFloatStateOf`, `mutableDoubleStateOf`. This is free performance — no reason to skip it.

## Avoid `remember` Pitfalls That Leak

- `remember { mutableStateOf(expensiveFn()) }` calls `expensiveFn()` on every recomposition. `mutableStateOf` is not lazy. Use `remember { mutableStateOf(expensiveFn()) }` only when `expensiveFn` is cheap; otherwise wrap: `remember { mutableStateOf(null) }.also { if (it.value == null) it.value = expensiveFn() }` — or hoist into the ViewModel, which is usually the real answer.
- `remember { scope.launch { ... } }` does not cancel on disposal. Use `rememberCoroutineScope().launch` from an event or `LaunchedEffect` for composition-driven work.

## Lambdas In Composables

With SSM on, most lambdas are compiler-memoized. You do **not** need to manually wrap every callback in `remember { { ... } }`. That pattern is legacy and adds noise.

Two cases where manual remembering still matters:

1. **Lambdas passed across module boundaries to generic composables whose generics hide the type.** The compiler can miss memoization. If a profiler shows identity-based churn, `remember`.
2. **Expensive derivations inside lambdas.** If the lambda itself is cheap but allocates a large structure, that allocation happens on every call. Move the allocation outside.

## Expensive Work In Composition

Composition runs often. Keep the body cheap.

Anti-patterns to flag:

- `file.readText()`, network calls, DB queries directly in a composable body.
- Heavy `list.filter { ... }.sortedBy { ... }.groupBy { ... }` chains inside a composable. Move to the ViewModel, or wrap in `remember(input)` if genuinely UI-local.
- `LocalConfiguration.current.screenWidthDp` / `LocalDensity.current.density` read inside a hot loop. Read once, pass the computed value.
- `stringResource(R.string.x, dynamicArg)` — fine, but allocates a new string every recomposition. If `dynamicArg` is stable, you can `remember(dynamicArg) { stringResource(...) }` — often not worth it, but worth knowing.

## Animations

From `references/effects.md` — two core rules:

- Target-driven: `LaunchedEffect(target) { animatable.animateTo(target) }`, not `scope.launch`.
- Per-frame reads: lambda-form modifiers (`Modifier.offset { ... }`, `Modifier.graphicsLayer { ... }`).

For `Crossfade` vs `AnimatedContent`:

- `Crossfade` is fine for standard fades between mutually exclusive content.
- Switch to `AnimatedContent` when you want custom enter/exit transitions, size-aware swaps, or different directions based on target.

## Canvas And Draw

- Prefer `Modifier.drawBehind { ... }` and `Modifier.drawWithCache { ... }` over a full `Canvas` composable for simple decorations. `drawWithCache` caches draw-only state between frames.
- Never read `LocalDensity.current` inside `drawBehind { ... }` — grab the density outside.
- Avoid reading state inside `drawBehind` unless you need it — reads there invalidate draw, which is cheaper than composition but still not free.

## Baseline Profiles

Shipping a baseline profile is still one of the biggest end-user performance wins a Compose app can get. If the module generates one, the `baseline-prof.txt` lives under `src/main/`. If not, flag as a suggested improvement rather than a required fix — generation needs a benchmark module.

## Grep Triggers

- `mutableStateOf<(Int|Long|Float|Double)>` — typed-primitive miss
- `remember\s*\{\s*mutableStateOf\s*\(\s*\w+\s*\)\s*\}` — parameter-seeded state (usually a bug — see `state.md`)
- `Modifier\.offset\(` / `Modifier\.alpha\(` / `Modifier\.scale\(` / `Modifier\.rotate\(` — look at the argument; if it reads an animated state, recommend lambda-form
- `items\(\s*\w+\s*\)\s*\{` in a `Lazy*` without `key =` — probably missing keys
- `animateItemPlacement\(` — migrate to `animateItem()`
- `@NonSkippableComposable` / `@DontMemoize` — demand justification

## Primary Sources

- `https://developer.android.com/develop/ui/compose/performance/bestpractices`
- `https://developer.android.com/develop/ui/compose/performance/stability`
- `https://developer.android.com/develop/ui/compose/performance/stability/strongskipping`
- `https://developer.android.com/develop/ui/compose/performance/phases`
- `https://developer.android.com/develop/ui/compose/lists` (keys and contentType)
- `https://developer.android.com/develop/ui/compose/performance/baseline-profiles`
