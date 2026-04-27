# Flows

`concurrency.md` covers **where** Flows live (lifecycle, scopes, dispatchers, cancellation). This file covers **which operator to pick and which Flow type to expose**. The two read together. LLMs reach for the wrong `flatMap`, forget `initialValue` on `stateIn`, expose `MutableStateFlow`, and use `combine` where `merge` was the right call. The fixes are mechanical once the shape is named.

## `StateFlow` vs `SharedFlow` vs Cold `Flow`

| You have… | Use | Notes |
|---|---|---|
| Continuously-valid state the UI renders (a current screen state, a logged-in user) | `StateFlow<T>` | Hot, conflated, always has a `value`, distinct-until-changed by default. |
| One-shot or replayable events with no "current value" (snackbar requests, navigation commands, in-app purchase results) | `SharedFlow<T>` | Hot, configurable at creation time. Has no `value`. |
| A description of work to perform on subscribe (a database query, a network request, a sensor stream) | cold `Flow<T>` | Re-runs from scratch per collector. The default in libraries (Room, Retrofit, DataStore). |

**Rules:**

- A ViewModel's UI state is almost always a `StateFlow`. Transient messages or navigation decisions that must survive lifecycle gaps should usually be represented in state and acknowledged by the UI. Ephemeral single-collector signals can use a `Channel<Event>` exposed as `Flow` via `receiveAsFlow()` (see `concurrency.md`); use `SharedFlow` when multiple subscribers must each observe the same event or when you deliberately need replay/buffer configuration.
- Cold flows are not bad — most repository APIs return them. The mistake is exposing a cold flow as a screen's state without first warming it via `stateIn` so that configuration changes don't trigger a fresh upstream subscription.
- `StateFlow` is `distinctUntilChanged` by default (using `equals`). If you wrap a non-data-class type, define `equals` properly or you'll either drop intended emissions (overly equal) or recompose constantly (always unequal).

## `stateIn` — The Default ViewModel Pattern

Convert a cold flow into a `StateFlow` the UI can collect with `collectAsStateWithLifecycle()`:

```kotlin
class FeedViewModel(repo: FeedRepository) : ViewModel() {
    val state: StateFlow<FeedState> = repo.feedStream()
        .map(FeedState::Loaded)
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = FeedState.Loading,
        )
}
```

**Rules:**

- `started = WhileSubscribed(5_000)` is the default recommendation for ViewModel UI state. Start the upstream when the first collector arrives, keep it alive for 5 seconds after the last collector leaves, then tear it down. Five seconds is the sweet spot for surviving configuration changes (rotation, theme switch, a transient backstack swap) without keeping the upstream alive in the background.
- `initialValue` is required by the API. Pick one that matches the screen's loading semantics. Do not default to `null` and `?.let { ... }` everywhere — model loading explicitly with a sealed type or a dedicated `Loading` state object.
- `Eagerly` only when the upstream is cheap and the value is needed immediately for the first frame (rare). `Lazily` only when you need the upstream to start on first subscription and never tear down (almost never the right call in a ViewModel).
- `WhileSubscribed(0)` tears the upstream down the instant the last collector leaves — useful for very expensive sources where you cannot tolerate a 5s grace period, but a frequent cause of refetch storms during configuration change.

**LLM tell:** `stateIn(viewModelScope)` with no `started` / `initialValue` arguments. That is the suspending overload: it waits for the first upstream emission and throws if the upstream never emits. It is not the ViewModel property-initializer shape. For screen state, use the non-suspending overload and pass `scope`, `started`, and `initialValue` explicitly.

## `shareIn` — When You Need a Hot `SharedFlow`

Use when multiple collectors should observe the same upstream and you need either:

- a configurable `replay` (so late subscribers see the last N values), or
- the absence of a "current value" (no need for `StateFlow.value`).

```kotlin
class LocationStream(scope: CoroutineScope, source: Flow<Location>) {
    val updates: SharedFlow<Location> = source
        .shareIn(
            scope = scope,
            started = SharingStarted.WhileSubscribed(5_000),
            replay = 1, // last known location for late subscribers
        )
}
```

**Rules:**

- `shareIn` takes only `scope`, `started`, and `replay`. Do not pass `extraBufferCapacity` or `onBufferOverflow` to `shareIn`; those are `MutableSharedFlow` constructor parameters.
- Configure `shareIn` buffering by placing `.buffer(capacity)` or `.conflate()` immediately before `.shareIn(...)`. The official docs describe this as overriding the sharing coroutine's default buffer.
- If `replay = 1` and a default value exists, prefer `stateIn` — its `value` accessor and conflation are usually what you actually want.

### `MutableSharedFlow` Event Streams

Use `MutableSharedFlow` directly when the ViewModel owns an event stream and must emit into it:

```kotlin
private val _events = MutableSharedFlow<UiEvent>(
    extraBufferCapacity = 1,
    onBufferOverflow = BufferOverflow.DROP_OLDEST,
)
val events: SharedFlow<UiEvent> = _events.asSharedFlow()
```

**Rules:**

- `extraBufferCapacity` and `onBufferOverflow` belong here, on `MutableSharedFlow(...)`, not on `shareIn(...)`.
- `tryEmit` exists on `MutableSharedFlow`, not on the read-only `SharedFlow` you expose publicly.
- Use `DROP_OLDEST` only when stale events are acceptable. For must-deliver work, use a suspending `emit`, a `Channel`, or persistent state instead.

## `flatMap` Variants — Cancellation Semantics

The single largest LLM error is reaching for plain `flatMap` (which Flow does not have under that name) or for `flatMapMerge` when `flatMapLatest` was intended.

| Operator | Behavior on new upstream emission |
|---|---|
| `flatMapLatest` | **Cancel** the in-flight inner flow. Subscribe to the new one. |
| `flatMapMerge` | Subscribe to the new inner flow concurrently. Merge emissions. |
| `flatMapConcat` | Wait for the current inner flow to complete. Then subscribe. |

```kotlin
// Search-as-you-type: every keystroke cancels the previous query
val results: Flow<List<Result>> = queryFlow
    .debounce(250.milliseconds)
    .distinctUntilChanged()
    .flatMapLatest { query -> repo.search(query) }
```

**Rules:**

- `flatMapLatest` is the right pick anytime a new input invalidates work-in-progress — search, filter, route changes, user-id swaps.
- `flatMapMerge` keeps every inner flow alive — useful for a fan-out where every emission spawns a parallel pipeline that should all reach the collector.
- `flatMapConcat` serializes — useful for ordered effects (apply migrations one at a time) but a deadlock risk if the inner flow never completes.

**LLM tell:** `flatMapLatest` used on a `StateFlow` of a single user id where you actually want to cancel a previous *animation*, not a previous *query*. If the inner flow doesn't produce values, you don't need a flatMap variant — use `collectLatest` directly.

## `combine`, `merge`, `zip`

```kotlin
// combine — emits the latest tuple whenever any input emits
val state: Flow<UiState> = combine(userFlow, settingsFlow, networkFlow) { user, settings, network ->
    UiState(user, settings, network.online)
}

// merge — interleaves emissions from any source, no pairing
val notifications: Flow<Notification> = merge(pushFlow, inAppFlow, scheduledFlow)

// zip — pair-wise; completes when the shortest input completes
val pairs: Flow<Pair<A, B>> = leftFlow.zip(rightFlow) { a, b -> a to b }
```

**Rules:**

- `combine` waits for *every* input to emit at least once before producing its first value. If one source is a cold flow that never emits, the combined flow never emits — a common screen-stuck-on-loading bug.
- `merge` is order-preserving per source but interleaved across sources. Do not rely on cross-source ordering.
- `zip` is rarely what you want for UI; pair-wise consumption usually models a request/response shape better solved with `flatMapLatest`.

**LLM tell:** `combine(a, b) { x, y -> Pair(x, y) }` is `a.zip(b) { x, y -> x to y }` if you actually want pair-wise. If you want "latest of each," `combine` is correct but the destructuring `combine(a, b) { (x, y) -> ... }` does **not** compile — you need named parameters: `combine(a, b) { x, y -> ... }`.

## `distinctUntilChanged`

`StateFlow` is already `distinctUntilChanged` by structural equality. Cold flows are not.

```kotlin
// Sensor source — emits identical readings often
sensorFlow
    .map { it.normalize() }
    .distinctUntilChanged()
    .collect { render(it) }
```

**Rules:**

- Apply just before the collector that cares about distinctness. Earlier in the pipeline you may want every emission for `scan`, throttling, or analytics.
- Custom equality: `distinctUntilChanged { old, new -> old.id == new.id }` when full-object equality is too strict.
- Do not chain `distinctUntilChanged` after a `StateFlow` you exposed via `stateIn` — it is a no-op, and signals confusion about which flow type you are working with.

## `debounce`, `sample`, And Click Throttling

UX-shaped rate limiters.

| Operator | Behavior |
|---|---|
| `debounce(d)` | Wait `d` of silence before emitting the latest value. Drop intermediates. Search-as-you-type. |
| `sample(d)` | Emit the latest value every `d` ticks. Polling-style throttling. |

```kotlin
// Search input
queryFlow.debounce(250.milliseconds)

// Scroll position read for analytics
scrollFlow.sample(1.seconds)
```

**Rules:**

- `debounce` is for "wait until the user stops." `sample` is for "snapshot every N." Mixing them up either spams the backend or feels laggy.
- `kotlinx.coroutines.flow` does not ship a `throttleLatest` / `throttleFirst` operator. If click throttling is needed, implement it deliberately in the UI event path or use a small, tested project-local extension — do not invent an import from `kotlinx.coroutines.flow`.

## Error Handling: `catch`, `retry`, `retryWhen`, `onCompletion`

```kotlin
val state: StateFlow<FeedState> = repo.feedStream()
    .map(FeedState::Loaded)
    .retryWhen { cause, attempt ->
        if (cause is IOException && attempt < 3) {
            delay(1.seconds * (1 shl attempt.toInt())) // exponential backoff
            true
        } else {
            false
        }
    }
    .catch { emit(FeedState.Error(it)) }
    .stateIn(viewModelScope, WhileSubscribed(5_000), FeedState.Loading)
```

**Rules:**

- `catch` only catches errors **upstream** of where it is placed. Put it after the operators whose failures you want to handle, and before any operator whose failures must propagate (or be caught by a different `catch` placed lower).
- `catch` does not catch exceptions thrown inside the downstream collector. Wrap collector-side imperative code in `try/catch` or move the work upstream into `map { }` so `catch` sees it.
- `retry` resubscribes to the upstream — fine for cold flows, redundant on `StateFlow` (which never completes).
- Use `retryWhen` for backoff strategies; the predicate gets `cause` and `attempt` count.
- `onCompletion { cause -> ... }` runs on success, error, **and** cancellation. It is the idiomatic place to log "stream ended" or to release resources held by the collector. `cause == null` distinguishes success from failure or cancellation; `cause is CancellationException` distinguishes cancellation specifically.

## Backpressure: `buffer`, `conflate`, `collectLatest`

When the producer is faster than the collector.

| Operator | Behavior |
|---|---|
| `buffer(n)` | Decouple producer and collector with an `n`-sized buffer. |
| `conflate` | Drop intermediate values; the collector always sees the latest. |
| `collectLatest { ... }` | Cancel the previous collector body when a new value arrives. |

```kotlin
// UI driven from a fast source — only the latest matters
heavyComputationFlow
    .conflate()
    .collect { render(it) } // skips intermediates if rendering is slow

// Each emission triggers async work that should be cancelled if a new emission arrives
imageRequests.collectLatest { request ->
    val bitmap = decoder.decode(request) // suspending; cancelled on next emission
    show(bitmap)
}
```

**Rules:**

- `conflate` is shorthand for `buffer(capacity = 0, onBufferOverflow = BufferOverflow.DROP_OLDEST)` / `Channel.CONFLATED`. Useful for sensor / scroll / animation values where stale frames are wasted work.
- `collectLatest` cancels the **collector body**, not the upstream. Use when each emission triggers cancellable work (a network call, an animation, a database write whose result is now stale).
- Do not use `collectLatest` on a flow that completes one emission per tick — there is nothing to cancel and you are paying for the structured concurrency overhead.

## Exposing `MutableStateFlow` Safely

```kotlin
class ProfileViewModel : ViewModel() {
    private val _state = MutableStateFlow<ProfileState>(ProfileState.Loading)
    val state: StateFlow<ProfileState> = _state.asStateFlow()

    private val _events = Channel<UiEvent>(Channel.BUFFERED)
    val events: Flow<UiEvent> = _events.receiveAsFlow()
}
```

**Rules:**

- The `_` prefix on the private mutable backing field plus a public read-only projection is the conventional shape. `asStateFlow()` is a free type-narrowing — it does not allocate a new flow.
- Mutating the public `state` field from outside the ViewModel is then a compile error, not a runtime convention.
- Same shape for `Channel` → `Flow` via `receiveAsFlow()`. Never expose a `Channel` directly.

## Common LLM Mistakes

- **Exposing `MutableStateFlow` publicly.** Use the private/`asStateFlow()` shape above. The audit skill flags this; the compose-agent should fix it.
- **Calling `stateIn(scope)` when exposing ViewModel UI state.** That is the suspending overload, not the usual property-initializer shape. Use `stateIn(scope, started, initialValue)` and pass `WhileSubscribed(5_000)` unless you have a documented reason to choose otherwise.
- **Omitting `initialValue` on `stateIn`.** The compiler will catch it on the three-argument overload, but LLMs sometimes write `.stateIn(scope, started = WhileSubscribed(5_000))` and add `?` to the type to compensate. Model loading explicitly instead.
- **Using bare `flatMap` (= `flatMapConcat`) where `flatMapLatest` is needed.** Search inputs, route changes, user-id swaps — all want cancellation-on-new-input.
- **`flow.collect { state.value = it }` inside a composable body.** Use `collectAsStateWithLifecycle()`. The composable body should never `collect`.
- **`combine(a, b) { (x, y) -> ... }`** does not compile. Use `combine(a, b) { x, y -> ... }`.
- **`combine` waiting forever** because one of the inputs never emitted. Give every cold input an `onStart { emit(initial) }` or use a `StateFlow` with a known initial value as the input.
- **Putting `catch` before the operator whose failures it should handle.** `catch` is upstream-only; place it after the risky operator.
- **`retry` on a hot `StateFlow`.** Hot flows do not complete, so `retry` is a no-op. Apply retry to the cold upstream before `stateIn`.
- **`launchIn` on a UI-tied scope.** `launchIn(scope)` is hard to read and easy to leak. Inside a composable, prefer `LaunchedEffect(Unit) { flow.collect { ... } }`. Inside a ViewModel, write `viewModelScope.launch { flow.collect { ... } }` explicitly.

## Primary Sources

- `https://kotlinlang.org/docs/flow.html`
- `https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/`
- `https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-shared-flow/`
- `https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/state-in.html`
- `https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/share-in.html`
- `https://developer.android.com/kotlin/flow`
- `https://developer.android.com/topic/architecture/ui-layer/state-production`
- `https://elizarov.medium.com/cold-flows-hot-channels-d74769805f9` (Roman Elizarov — cold flows, hot channels)
