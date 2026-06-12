# Animation

Compose has one of the largest animation API surfaces in the framework, and LLMs pick the wrong tool from it constantly. The mistakes are not random — they cluster. This reference targets the clusters: wrong API for the job, imperative animation in the wrong place, missing `remember`, `tween`-everything, and animations that recompose every frame instead of reading deferred.

Animation is not a separate concern from the rest of this skill. A bad animation is usually a state bug (`Animatable` not remembered), an effect bug (`scope.launch` in composition), or a performance bug (non-deferred read) wearing an animation costume. Cross-reference `effects.md`, `state.md`, and `performance.md` freely.

## Choosing the Right API

This is the **first** decision, and the one LLMs get wrong most often — they reach for `Animatable` + `LaunchedEffect` for everything, including cases that need no coroutine and no effect at all.

| You want to… | Use | Coroutine? |
|---|---|---|
| animate a single value when state changes (size, color, offset, alpha, dp) | `animate*AsState` | No |
| animate several properties off the **same** state change, in sync | `updateTransition` + `transition.animate*` | No |
| imperatively drive a value — gesture follow, fling, sequence, cancel/restart | `remember { Animatable(...) }` | Yes |
| loop forever (pulsing, shimmer, indeterminate) | `rememberInfiniteTransition` | No |
| enter/exit a composable from the tree | `AnimatedVisibility` | No |
| swap between different content based on target state | `AnimatedContent` (or `Crossfade` for a plain fade) | No |
| animate layout size changes automatically | `Modifier.animateContentSize()` | No |
| animate item position in a lazy list | `Modifier.animateItem()` (in `LazyColumn`/`LazyRow` item scope) | No |

**LLM tell:** any `val anim = remember { Animatable(...) }` paired with a `LaunchedEffect` that just calls `anim.animateTo(target)` whenever `target` changes. That is `animateXAsState` rewritten in 6 lines with a bug surface. Collapse it.

```kotlin
// LLM-generated — imperative animation for a declarative job
val offset = remember { Animatable(0f) }
LaunchedEffect(expanded) {
    offset.animateTo(if (expanded) 200f else 0f)
}
Box(Modifier.offset { IntOffset(offset.value.roundToInt(), 0) })

// Correct — declarative, no coroutine, no effect, interruptible by default
val offset by animateFloatAsState(
    targetValue = if (expanded) 200f else 0f,
    label = "offset",
)
Box(Modifier.offset { IntOffset(offset.roundToInt(), 0) })
```

Reach for `Animatable` only when you genuinely need imperative control: snapping to a gesture position, decay/fling, or running a multi-step sequence (`animateTo` then `animateTo` then `snapTo`) that no single target can express.

<https://developer.android.com/develop/ui/compose/animation/choose-api>

## `Animatable` Must Be `remember`ed

A bare `Animatable(...)` in a composable body is recreated on every recomposition, resetting the animation mid-flight.

```kotlin
// Wrong — new Animatable each recomposition, animation restarts/jumps
val alpha = Animatable(1f)

// Right
val alpha = remember { Animatable(1f) }
```

This is the single most common animation bug in generated code. It is a `state.md` violation (`remember` for identity stability) that happens to involve an animation. Same rule as `rememberCoroutineScope`, `LazyListState`, etc.: a stable object owned by the call site is created inside `remember`. If the composable can be re-keyed by a parent (`key(id) { ... }`), the `Animatable` resets with it — usually what you want.

<https://developer.android.com/develop/ui/compose/animation/value-based#animatable>

## Animations Are Target-Driven, Launched in `LaunchedEffect` — Never `scope.launch` in Composition

When you do need imperative `Animatable`, launch it from a `LaunchedEffect` keyed on the target, not from `rememberCoroutineScope().launch` inside the composition body and not from a `SideEffect`.

```kotlin
// Wrong — launching animation work during composition
val scope = rememberCoroutineScope()
val offset = remember { Animatable(0f) }
scope.launch { offset.animateTo(target) }   // runs on every recomposition

// Right — keyed effect, cancels and restarts cleanly when target changes
val offset = remember { Animatable(0f) }
LaunchedEffect(target) {
    offset.animateTo(target, animationSpec = spring())
}
```

`rememberCoroutineScope().launch` is for animations triggered by a **user event** (inside an `onClick`, a drag handler, a gesture `detectTapGestures`) — not for reacting to state. If the animation should follow a state value, that is `LaunchedEffect(state)` or, better, `animate*AsState`. See `effects.md` "LaunchedEffect — The Default Effect".

<https://developer.android.com/develop/ui/compose/animation/value-based#animatable>

## `updateTransition` — One Clock for Properties That Move Together

When several visual properties should animate off the **same** state flip — expanded/collapsed, selected/unselected, loading/loaded — LLMs often emit a stack of independent `animate*AsState` calls. Each runs on its own clock, so alpha, scale, and color can finish at different times and feel sloppy.

Use one `updateTransition(targetState)` and derive each property with `transition.animateFloat`, `transition.animateColor`, `transition.animateDp`, etc. They share timing, retarget together, and each child animation still gets its own `label`.

```kotlin
// LLM-generated — three separate animations, no shared clock, can drift
val alpha by animateFloatAsState(if (expanded) 1f else 0.5f, label = "alpha")
val scale by animateFloatAsState(if (expanded) 1f else 0.8f, label = "scale")
val color by animateColorAsState(if (expanded) Blue else Gray, label = "color")

// Correct — one transition, properties stay in sync off the same target
val transition = updateTransition(targetState = expanded, label = "expandCard")
val alpha by transition.animateFloat(label = "alpha") { if (it) 1f else 0.5f }
val scale by transition.animateFloat(label = "scale") { if (it) 1f else 0.8f }
val color by transition.animateColor(label = "color") { if (it) Blue else Gray }
```

Still read the animated outputs in the layout/draw phase (`Modifier.graphicsLayer { scaleX = scale; alpha = alpha }`), same deferred-read rule as single-value animations.

If only **one** property tracks the state, `animate*AsState` is still the right tool — do not introduce `updateTransition` for a single value.

<https://developer.android.com/develop/ui/compose/animation/value-based#updatetransition>

## `spring` by Default, `tween`/`keyframes` Only for Deterministic Timelines

LLMs default to `tween(durationMillis = 300)` everywhere. The framework default and the recommended choice for interaction-driven motion is `spring()`: it is physically continuous and **interruptible** — if the target changes mid-animation, a spring retargets smoothly from its current velocity, while a `tween` restarts from zero and visibly stutters.

| Spec | Use for |
|---|---|
| `spring()` (default) | gestures, state-driven UI motion, anything that can be interrupted/retargeted |
| `tween(durationMillis, easing)` | brand-timed motion, choreographed sequences where exact duration matters |
| `keyframes { }` | multi-stage motion with intermediate values at specific times |
| `snap()` | jump instantly (e.g. when reduced-motion is requested) |

```kotlin
// Smells of a magic number and a non-interruptible curve for an interactive value
animateDpAsState(target, animationSpec = tween(300), label = "x")

// Interruptible, retargets from current velocity
animateDpAsState(target, animationSpec = spring(stiffness = Spring.StiffnessMedium), label = "x")
```

Tune springs with `dampingRatio` / `stiffness` constants (`Spring.DampingRatioMediumBouncy`, `Spring.StiffnessLow`), not raw floats, unless you have a measured reason.

<https://developer.android.com/develop/ui/compose/animation/customize#animationspec>

## Always Pass `label`

`animate*AsState`, `updateTransition`, and `transition.animate*` all take a `label`. Omitting it gives the Animation Inspector a useless generic name and is a constant lint-level miss in generated code. Label every animation with what it animates.

```kotlin
// Before
val color by animateColorAsState(if (selected) Blue else Gray)

// After
val color by animateColorAsState(if (selected) Blue else Gray, label = "selectionColor")
```

<https://developer.android.com/develop/ui/compose/tooling/animation-preview>

## Defer Animated Reads to Layout/Draw — Don't Recompose Every Frame

An animation produces a new value every frame (~16 ms). If that value is read during **composition**, the composable recomposes on every frame for the whole animation. Read it in the **layout** or **draw** phase instead, via the lambda form of the modifier. This is the same deferred-read rule as `performance.md`, and animation is where it bites hardest.

```kotlin
val dx by animateDpAsState(if (open) 0.dp else (-240).dp, label = "drawerX")

// Wrong — value read in composition; recomposes every frame of the animation
Box(Modifier.offset(x = dx))

// Right — read in layout phase; skips recomposition entirely
Box(Modifier.offset { IntOffset(dx.roundToPx(), 0) })
```

Same idea for the others:

- alpha → `Modifier.graphicsLayer { alpha = animatedAlpha }`, not `Modifier.alpha(animatedAlpha)`
- rotation / scale / translation → `Modifier.graphicsLayer { ... }`
- padding / margin-like insets → do not pipe an animated `Dp` into `Modifier.padding(animatedDp)`; prefer `Modifier.offset { ... }` or `Modifier.graphicsLayer { translationX = ... }` for animated spacing
- background color → `Modifier.drawBehind { drawRect(animatedColor) }` when it's purely visual

If you animate a `Color` and only use it as a background/tint, prefer a draw-phase read over recomposing the subtree.

<https://developer.android.com/develop/ui/compose/performance/bestpractices#defer-reads>

## `AnimatedContent` — Give It a `contentKey` and a Deliberate `transitionSpec`

Two recurring `AnimatedContent` mistakes:

1. **No `contentKey`.** `AnimatedContent` re-runs its transition whenever `targetState` is not equal to the current state. If `targetState` is a big data class, an unrelated field change triggers a spurious content animation. Provide `contentKey = { it.id }` (or the field that actually defines "different content") so it only animates on meaningful changes.

2. **Default `transitionSpec` left in place.** The default fades **and** animates size with a `SizeTransform`, which makes surrounding layout jump. Set an explicit `transitionSpec` and decide whether you want `SizeTransform(clip = false)` or none.

```kotlin
AnimatedContent(
    targetState = uiState,
    contentKey = { it.screen },                 // only animate when the screen actually changes
    transitionSpec = {
        (fadeIn() + slideInVertically { it / 2 }) togetherWith fadeOut()
    },
    label = "screenSwap",
) { state ->
    when (state.screen) { /* ... */ }
}
```

Use `Crossfade` instead when all you want is a plain fade between two states — it is the simpler, harder-to-misuse tool.

<https://developer.android.com/develop/ui/compose/animation/composables-modifiers#animatedcontent>

## `AnimatedVisibility` — Match Enter/Exit, and Don't Re-`remember` MutableTransitionState

For simple appear/disappear, pass a `Boolean`. For "animate the first appearance too" or to observe the running transition, use `remember { MutableTransitionState(false) }` — and it must be remembered, same rule as `Animatable`.

```kotlin
val state = remember { MutableTransitionState(false) }.apply { targetState = true }
AnimatedVisibility(visibleState = state, enter = fadeIn() + expandVertically(), exit = fadeOut()) { /* ... */ }
```

Enter and exit transitions should be deliberate counterparts. `expandVertically()` in / `shrinkVertically()` out reads as one motion; `expandVertically()` in / `fadeOut()` out reads as two unrelated ones unless that's intentional.

Children inside `AnimatedVisibility` that need their own enter/exit should use `Modifier.animateEnterExit()` in the `AnimatedVisibilityScope`, not a separate outer `AnimatedVisibility` or hand-rolled `animate*AsState` tied to the same boolean.

<https://developer.android.com/develop/ui/compose/animation/composables-modifiers#animatedvisibility>

## `animateContentSize()` vs Manual Animation

When a composable's content changes size and you just want the size change to animate smoothly, `Modifier.animateContentSize()` is the one-liner — no state, no `AnimatedContent`. Reach for `AnimatedContent` only when the **content itself** swaps (different children), not when the same content merely grows or shrinks.

```kotlin
Text(text, Modifier.animateContentSize())   // expandable text, "read more"
```

Order matters: place `animateContentSize()` before explicit size modifiers such as `size`, `height`, `width`, or `defaultMinSize`, so the modifier reports the animated size change to layout. Keep ordinary visual modifiers (`background`, `clip`, borders) ordered deliberately around the animated box. See `modifiers.md` on ordering.

<https://developer.android.com/develop/ui/compose/animation/composables-modifiers#animatedcontentsize>

## Lazy Lists — `animateItem()`, Not Hand-Rolled

For reorder/insert/remove animations in `LazyColumn`/`LazyRow`, use `Modifier.animateItem()` inside the item scope, and make sure the list has stable `key =`s — the animation is keyed off them. Hand-rolled offset animations on list items fight the lazy layout and are a frequent generated-code mistake.

```kotlin
LazyColumn {
    items(rows, key = { it.id }) { row ->
        RowCard(row, Modifier.animateItem())
    }
}
```

(`animateItem()` replaced the experimental `animateItemPlacement()` — flag the old name if you see it. See `api.md`.)

<https://developer.android.com/develop/ui/compose/lists#item-animations>

## Infinite Animations Must Stop When Off-Screen

`rememberInfiniteTransition` keeps producing frames as long as it is in composition. A shimmer or pulse that stays composed but scrolled off-screen, or left running on a screen that is no longer visible, burns frames and battery. Two guards:

- Don't compose the animating node when it isn't visible (let the lazy layout dispose it, or gate it behind the state that needs it).
- For "animate only while STARTED", drive the animation from a value gated on lifecycle rather than an always-on infinite transition.

This is the animation-shaped version of the `collectAsStateWithLifecycle` battery rule in `concurrency.md`: don't do continuous work for a screen the user can't see.

<https://developer.android.com/develop/ui/compose/animation/value-based#rememberinfinitetransition>

## Gesture-Driven Motion — `snapTo` While Dragging, `animateDecay` / `animateTo` on Release

This is the case where `Animatable` earns its keep. During a drag, follow the finger with `snapTo` (or `updateBounds` when clamping). On release, run `animateDecay` for fling or `animateTo` for snap-back — launched from the gesture callback via `rememberCoroutineScope()`, not from composition.

```kotlin
val offsetX = remember { Animatable(0f) }
val scope = rememberCoroutineScope()

Box(
    Modifier
        .offset { IntOffset(offsetX.value.roundToInt(), 0) }
        .pointerInput(Unit) {
            detectHorizontalDragGestures(
                onDrag = { change, dragAmount ->
                    change.consume()
                    scope.launch { offsetX.snapTo(offsetX.value + dragAmount) }
                },
                onDragEnd = {
                    scope.launch {
                        offsetX.animateDecay(
                            initialVelocity = offsetX.velocity,
                            animationSpec = exponentialDecay(),
                        )
                    }
                },
            )
        },
)
```

Do not use `LaunchedEffect` to chase a drag position — that is event-driven, not target-driven. Do not use `animate*AsState` for continuous finger tracking.

<https://developer.android.com/develop/ui/compose/animation/value-based#animatable>

## Reduced Motion — Respect System Duration Scale

Compose animation coroutines run with a platform `MotionDurationScale`. When that scale is `0f`, duration-based motion finishes on the next frame callback. Do not invent a `LocalAccessibilityManager.isReduceMotionEnabled` flag — that is not a Compose UI API. For normal `animate*AsState`, `updateTransition`, and `Animatable` motion, rely on the framework duration scale instead of hand-reading Settings.

```kotlin
LaunchedEffect(target) {
    val durationScale = coroutineContext[MotionDurationScale]?.scaleFactor ?: 1f
    if (durationScale == 0f) {
        offset.snapTo(target)
    } else {
        offset.animateTo(target, animationSpec = spring())
    }
}
```

For decorative motion that is not necessary to communicate state (shimmer, pulse, background flourish), expose an explicit `reduceMotion: Boolean` from your app/platform settings layer and do not compose the infinite transition when it is true. If a user asks for reduced-motion support and the project has no app-level setting yet, add the parameter seam rather than reaching for a nonexistent Compose local.

<https://developer.android.com/reference/kotlin/androidx/compose/ui/MotionDurationScale>

## Anti-Patterns Checklist

- `Animatable(...)` / `MutableTransitionState(...)` without `remember`.
- `Animatable` + `LaunchedEffect(target){ animateTo(target) }` where `animate*AsState` would do.
- `scope.launch { animateTo() }` in composition to react to state (should be `LaunchedEffect` or `animate*AsState`).
- `tween(<magic number>)` for interactive/interruptible motion instead of `spring()`.
- Missing `label` on `animate*AsState` / `updateTransition`.
- Animated value read in composition (`Modifier.offset(x)`, `Modifier.alpha(a)`, `Modifier.padding(animatedDp)`) instead of the lambda/`graphicsLayer` form.
- `animate*AsState` used for continuous gesture tracking — use `Animatable.snapTo` during drag instead.
- Decorative motion with no app-level reduced-motion seam on non-essential infinite/decorative animations.
- `AnimatedContent` with no `contentKey` and the default `SizeTransform`.
- Separate `animate*AsState` calls for properties that move together — use one `updateTransition`.
- `rememberInfiniteTransition` left running off-screen.
- Hand-rolled list item movement instead of `Modifier.animateItem()` with stable keys.

## Primary Sources

- <https://developer.android.com/develop/ui/compose/animation/choose-api>
- <https://developer.android.com/develop/ui/compose/animation/value-based>
- <https://developer.android.com/develop/ui/compose/animation/customize>
- <https://developer.android.com/develop/ui/compose/animation/composables-modifiers>
- <https://developer.android.com/develop/ui/compose/performance/bestpractices#defer-reads>
- <https://developer.android.com/develop/ui/compose/lists#item-animations>
- <https://developer.android.com/reference/kotlin/androidx/compose/ui/MotionDurationScale>
