# Side Effects

Composables must be side-effect free during composition. Every I/O, animation launch, analytics event, subscription, timer, snackbar show, scroll-to, or focus request belongs inside a side-effect API — never in the composable body.

This is the single largest category of LLM-generated Compose bugs.

## The Decision Tree

| You want to… | Use |
|---|---|
| run a suspending block when a key changes | `LaunchedEffect(key) { ... }` |
| run a suspending block exactly once while this composable is in composition | `LaunchedEffect(Unit) { ... }` |
| run **and** register cleanup (listener, observer, callback registration) | `DisposableEffect(key) { ... ; onDispose { ... } }` |
| publish composition values to a non-Compose system every commit | `SideEffect { ... }` |
| convert a callback / Flow / suspend source into Compose state | `produceState(initialValue, key) { ... }` |
| observe a Compose state change as a Flow inside a coroutine | `snapshotFlow { stateRead() }.collect { ... }` |
| launch a coroutine in response to a user event (click, drag, etc.) | `rememberCoroutineScope()` + `scope.launch { ... }` |
| use a callback inside a long-lived effect while keeping the latest version | `rememberUpdatedState(callback)` |

## The Keying Rule (Recap)

Same rule covers `LaunchedEffect`, `DisposableEffect`, `produceState`, `remember`, and the setup of a remembered `derivedStateOf`. Every changing value the effect or producer reads must be either:

1. listed in the key arguments,
2. intentionally fixed for the call-site lifetime (constant, stable owned object, initial-value capture), or
3. read through `rememberUpdatedState(value)` when the effect should keep running but observe the latest value.

Anything else captures stale values silently — the effect calls a yesterday-callback, the producer keeps subscribing to the wrong id, the listener fires into a closed-over old lambda. The full rule with examples lives in `state.md` under "The Keying Rule" — apply it identically here.

## `LaunchedEffect` — The Default Effect

```kotlin
@Composable
fun ProfileScreen(userId: String, viewModel: ProfileViewModel) {
    LaunchedEffect(userId) {
        viewModel.load(userId)
    }
}
```

Rules:

- The **keys** control restart. Change them → the current coroutine is cancelled and the block runs again.
- Use `LaunchedEffect(Unit)` only for genuinely one-shot work (log a screen view once, launch an initial animation). Never as a quick fix for "I want it to run on every recomposition" — that is a recomposition bug, not an effect pattern.
- If you catch yourself passing `key = coroutineScope` or similar as a key, you are doing it wrong. Keys are the inputs whose change should restart the work.
- Do **not** collect flows manually when the goal is "render this value"; expose the flow as state and call `collectAsStateWithLifecycle`. If you truly need a coroutine for events, key the effect on the changing flow identity: `LaunchedEffect(events) { events.collect { ... } }`. Use `LaunchedEffect(Unit)` only when the flow object is intentionally stable for the call-site lifetime.

**LLM tell:** using `LaunchedEffect` to read state that only gets read, not written to — that is a recomposition, not an effect.

## Lifecycle Effects — `LifecycleStartEffect` / `LifecycleResumeEffect`

From `androidx.lifecycle:lifecycle-runtime-compose` 2.8+. Purpose-built for "run X when the screen becomes STARTED / RESUMED, clean up when it leaves that state". Use these instead of hand-rolling `DisposableEffect` + `LifecycleEventObserver`.

```kotlin
// Wrong — 10 lines of boilerplate, easy to typo the observer
val lifecycleOwner = LocalLifecycleOwner.current
DisposableEffect(lifecycleOwner) {
    val observer = LifecycleEventObserver { _, event ->
        if (event == Lifecycle.Event.ON_START) viewModel.refresh()
    }
    lifecycleOwner.lifecycle.addObserver(observer)
    onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
}

// Right
LifecycleStartEffect(Unit) {
    viewModel.refresh()
    onStopOrDispose { /* cleanup when leaving STARTED, or when composable disposes */ }
}
```

Use `LifecycleResumeEffect` for work that should run only when the screen is actually RESUMED (foregrounded, user-interactable). Typical case: starting a camera preview, resuming a video, subscribing to a noisy sensor.

These effects are **strictly better** than the `DisposableEffect` + observer pattern whenever you only care about START/STOP or RESUME/PAUSE. Keep the manual `DisposableEffect` only when you need multiple specific lifecycle events (rare) or you are on an older lifecycle version.

## `DisposableEffect` — Registration With Cleanup

Use when the work registers something external (a broadcast receiver, a callback, a sensor listener, an observer on a non-Compose stream) that you must unregister when the composable leaves composition or when the key changes.

```kotlin
@Composable
fun OrientationSensor(
    onOrientationChanged: (Float) -> Unit,
) {
    val context = LocalContext.current
    val onChanged by rememberUpdatedState(onOrientationChanged)
    DisposableEffect(context) {
        val manager = context.getSystemService<SensorManager>()!!
        val listener = object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent) {
                onChanged(event.values[0])
            }
            override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) = Unit
        }
        manager.registerListener(listener, manager.getDefaultSensor(TYPE_ORIENTATION), SENSOR_DELAY_UI)
        onDispose { manager.unregisterListener(listener) }
    }
}
```

Rules:

- Every `DisposableEffect` must end with `onDispose { ... }`. It is a compile error to forget, but LLMs sometimes return code missing the `onDispose` block and then assume the compiler will be fine.
- The `onDispose` cleanup runs when the keys change (before the block re-runs) and when the composable leaves composition.
- Wrap the captured callback in `rememberUpdatedState` when the effect is keyed on something **other than** the callback. Otherwise the listener keeps calling the stale callback for the rest of the effect's life.

## `SideEffect` — Publishing To A Non-Compose System

Runs after every successful composition. Use for imperative calls into a non-Compose library that always need the latest composition value.

```kotlin
@Composable
fun AnalyticsTracker(screenName: String) {
    val analytics = LocalAnalytics.current
    SideEffect { analytics.setCurrentScreen(screenName) }
}
```

Rules:

- Do not put suspending work in `SideEffect` — it is synchronous. If you need suspending work, you want `LaunchedEffect`.
- Do not start state mutations in `SideEffect` that could re-trigger composition. It is intended as a one-way bridge **out** of Compose.

## `produceState` — Callback / Flow / Suspend → `State<T>`

Adapts an external source into a `State<T>` that composables can read directly.

```kotlin
@Composable
fun imageState(url: String, loader: ImageLoader): State<ImageState> {
    return produceState<ImageState>(initialValue = ImageState.Loading, url, loader) {
        value = try {
            ImageState.Loaded(loader.load(url))
        } catch (e: IOException) {
            ImageState.Error(e)
        }
    }
}
```

Rules:

- `produceState` launches a coroutine scoped to composition. Cancel-awareness is automatic.
- For callback-based producers, use `awaitDispose { ... }` inside to unregister on leave.
- Do not use `produceState` when the source is already a `StateFlow` — collect it with `collectAsStateWithLifecycle` instead.
- Apply the keying rule. Every changing value the producer block reads must be a `produceState` key (`url`, `loader` above), or be wrapped with `rememberUpdatedState` if the producer must keep running across changes. A `produceState(initialValue) { ... }` with no extra keys but a body that reads parameters is the same stale-capture bug as a `LaunchedEffect(Unit)` reading a parameter.

## `snapshotFlow` — Compose State → `Flow<T>`

Observes reads of Compose state from inside a coroutine.

```kotlin
@Composable
fun InfiniteList(listState: LazyListState, loadMore: () -> Unit) {
    LaunchedEffect(listState) {
        snapshotFlow { listState.layoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0 }
            .distinctUntilChanged()
            .filter { it >= listState.layoutInfo.totalItemsCount - 3 }
            .collect { loadMore() }
    }
}
```

Rules:

- `snapshotFlow` emits whenever a tracked read changes between snapshots. Apply `distinctUntilChanged()` in most cases — the raw flow emits on every snapshot that touched the read, even if the value is equal.
- Only the state **read inside the lambda** is tracked. State captured by reference before the lambda runs is not tracked.

## `rememberCoroutineScope` — Event-Driven Work

Use for coroutines launched **in response to events**, not during composition.

```kotlin
@Composable
fun RefreshButton(onRefresh: suspend () -> Unit) {
    val scope = rememberCoroutineScope()
    Button(onClick = { scope.launch { onRefresh() } }) {
        Text("Refresh")
    }
}
```

Rules:

- `rememberCoroutineScope` gives you a scope tied to composition. Work launched here is cancelled when the composable leaves composition.
- Do not use it to kick off state-driven animations — that belongs in `LaunchedEffect(target)`.
- Never call `launch` directly in the composable body. It runs on every recomposition.

## `rememberUpdatedState` — Capture The Latest Lambda

When a long-lived effect captures a callback, wrap the callback so the effect always sees the latest version without restarting.

```kotlin
@Composable
fun HeartbeatTimer(
    period: Duration = 5.seconds,
    onTick: () -> Unit,
) {
    val currentOnTick by rememberUpdatedState(onTick)
    LaunchedEffect(period) {
        while (isActive) {
            delay(period)
            currentOnTick() // always the latest
        }
    }
}
```

Rules:

- Only use for values captured inside an effect whose keys do not include the value.
- If the value is a key, you do not need `rememberUpdatedState` — the effect already restarts with the new value.

## Animations Driven From Effects

Target-driven animations belong in `LaunchedEffect(target)`, not `rememberCoroutineScope().launch`:

```kotlin
// Wrong — launches a new animation on every click, no restart logic
val anim = remember { Animatable(0f) }
val scope = rememberCoroutineScope()
Button(onClick = { scope.launch { anim.animateTo(1f) } }) { ... }

// Right — every time `expanded` flips, the in-flight animation cancels and restarts
val anim = remember { Animatable(0f) }
LaunchedEffect(expanded) {
    anim.animateTo(if (expanded) 1f else 0f)
}
```

For value-based animations (`animate*AsState`), no effect is needed at all — the API is already effect-shaped internally.

## Anti-Patterns

- **Calling a suspend function directly in the composable body.** Does not compile, but LLMs sometimes wrap it in an IIFE or an `also { GlobalScope.launch { ... } }`. Both are wrong.
- **`GlobalScope.launch`** anywhere in UI code. Use the composition's scope or the ViewModel's scope.
- **State mutation in a `@Composable` function body outside `remember`/`mutableStateOf`.** The read–write cycle causes invalidation storms ("backwards writes").
- **Using `LaunchedEffect` where `collectAsStateWithLifecycle` suffices.** If the goal is "show this flow as state", do not open the flow manually.
- **Nesting effects.** A `LaunchedEffect` inside another `LaunchedEffect` is almost always a sign the outer effect should have had different keys.

## Primary Sources

- `https://developer.android.com/develop/ui/compose/side-effects`
- `https://developer.android.com/develop/ui/compose/lifecycle`
- `https://developer.android.com/topic/architecture/ui-layer/state-production`
