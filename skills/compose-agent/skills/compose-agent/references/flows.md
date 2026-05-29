# Flows

`concurrency.md` covers **where** Flows live (lifecycle, scopes, dispatchers, cancellation). This file covers **which operator to pick and which Flow type to expose**. The two read together. LLMs reach for the wrong `flatMap`, forget `initialValue` on `stateIn`, expose `MutableStateFlow`, and use `combine` where `merge` was the right call. The fixes are mechanical once the shape is named.

## Flow Operators Belong Outside The Composable Body

Flow operators in a `@Composable` body are a smell. The UI layer should consume **pre-digested screen state** — a `StateFlow<UiState>` (or equivalent) shaped by a presenter, state holder, or ViewModel. Pipelines composed in the render layer read concise but cost a lot:

- `combine`, `merge`, `flatMapLatest`, `debounce`, `stateIn`, `shareIn` etc. created inline are rebuilt on **every composition** unless wrapped in `remember(...)` with the right keys — and even then the data shape is in the wrong layer.
- The composable picks up subscription, error handling, and timing concerns that should live with the data.
- Previews, tests, and screenshot tooling lose any way to inject state — they would need a working coroutine pipeline.

```kotlin
// Wrong — pipeline rebuilt every recomposition; combine never emits until both sources do
@Composable
fun FeedScreen(repo: FeedRepository, settingsRepo: SettingsRepository) {
    val state by combine(repo.feed(), settingsRepo.settings()) { feed, settings ->
        FeedUiState(feed, settings)
    }.collectAsStateWithLifecycle(initialValue = FeedUiState.Loading)
    // ...
}

// Right — pipeline lives in the presenter, UI consumes a single StateFlow
class FeedViewModel(repo: FeedRepository, settingsRepo: SettingsRepository) : ViewModel() {
    val state: StateFlow<FeedUiState> = combine(repo.feed(), settingsRepo.settings()) { feed, settings ->
        FeedUiState.Loaded(feed, settings)
    }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), FeedUiState.Loading)
}

@Composable
fun FeedScreen(viewModel: FeedViewModel) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    // ...
}
```

**Two questions, separate answers:**

- **Where do you collect?** A screen-boundary `collectAsStateWithLifecycle()` (or a keyed `LaunchedEffect(flow) { flow.collect { ... } }` for events) is fine. Collection happens at the UI edge.
- **Where do you shape?** Operators (`combine`, `flatMapLatest`, `stateIn`, `debounce`, `retry`, etc.) belong in a presenter / state holder / ViewModel — **not** in a composable. The data layer hands the UI a single coherent stream.

**LLM tells:**

- `combine(...).collectAsStateWithLifecycle(...)` directly inside a composable.
- `repo.flow().map { ... }.stateIn(rememberCoroutineScope(), ...)` — `stateIn` from a composition-scoped scope rebuilds on every recomposition and tears down on disposal; this is almost never the right boundary.
- `flatMapLatest { ... }` chained inside an `@Composable` body.
- A `@Composable` with three or more Flow operator imports — the screen is doing presenter work.

## `StateFlow` vs `SharedFlow` vs Cold `Flow`

| You have… | Use | Notes |
|---|---|---|
| Continuously-valid state the UI renders (a current screen state, a logged-in user) | `StateFlow<T>` | Hot, conflated, always has a `value`, distinct-until-changed by default. |
| One-shot or replayable events with no "current value" (snackbar requests, haptic ticks, scroll commands, replayable fan-out notifications) | `SharedFlow<T>` | Hot, configurable at creation time. Has no `value`. |
| A description of work to perform on subscribe (a database query, a network request, a sensor stream) | cold `Flow<T>` | Re-runs from scratch per collector. The default in libraries (Room, Retrofit, DataStore). |

**Rules:**

- A ViewModel's UI state is almost always a `StateFlow`. Transient messages or navigation decisions that must survive lifecycle gaps should usually be represented in state and acknowledged by the UI. Ephemeral single-collector signals can use a `Channel<Event>` exposed as `Flow` via `receiveAsFlow()` (see `concurrency.md`); use `SharedFlow` when multiple subscribers must each observe the same event or when you deliberately need replay/buffer configuration.
- Cold flows are not bad — most repository APIs return them. The mistake is exposing a cold flow as a screen's state without first warming it via `stateIn` so that configuration changes don't trigger a fresh upstream subscription.
- `StateFlow` is `distinctUntilChanged` by default (using `equals`). If you wrap a non-data-class type, define `equals` properly or you'll either drop intended emissions (overly equal) or recompose constantly (always unequal).

### Prefer Durable State + Acknowledgement Over One-Shot Events

If losing the signal would desynchronize the UI from the underlying state, the signal is **not** ephemeral — model it as state plus an acknowledgement, not as a one-shot event.

```kotlin
// Wrong — purchase result is a one-shot event. If the screen is in STOPPED when emit
// happens, SharedFlow drops it (replay = 0) and the UI never reflects the success.
private val _events = MutableSharedFlow<PurchaseResult>()
val events: SharedFlow<PurchaseResult> = _events.asSharedFlow()

// Right — outcome is durable state; UI acknowledges to clear it.
data class CheckoutUiState(
    val cart: Cart,
    val pendingResult: PurchaseResult? = null,
)

fun resultAcknowledged() {
    _state.update { it.copy(pendingResult = null) }
}
```

Use ephemeral channels (`SharedFlow` with `replay = 0`, `Channel<Event>.receiveAsFlow()`) only for genuinely fire-and-forget UI commands where dropping is acceptable: a transient snackbar, a haptic tick, scroll-to-top after a refresh. Anything that influences "what the user thinks the app did" — payment outcomes, deletion confirmations, save success, navigation that must happen — belongs in state, with the UI clearing it after consumption.

**LLM tell:** every UI outcome modeled as a `MutableSharedFlow<UiEvent>`. The `UiEvent` sealed class grows; bugs appear after process death or backgrounding. Convert outcomes that must not be lost into state fields and add `onXxxConsumed` callbacks.

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
- Same shape for `MutableSharedFlow` → `SharedFlow` via `asSharedFlow()`. Same rule: keep the mutable end private.
- A presenter / state holder / ViewModel exposes one coherent `StateFlow<UiState>` per screen, not a fan-out of three `StateFlow`s the UI has to recombine. If you find the UI calling `combine(viewModel.a, viewModel.b, viewModel.c)`, the projection belongs upstream.

## Operator Selection Smells

A short index of operator-shape mistakes worth flagging on review. Each one has a shape it pretends to be and a shape it actually is.

| Smell | What's actually happening | Fix |
|---|---|---|
| `zip` used to build screen state | Pair-wise: blocks waiting for the slower side; first slow side starves the screen | Use `combine` with a `StateFlow` (or `onStart { emit(initial) }`) per input |
| `flatMapMerge` where new input invalidates old work | Old in-flight work keeps running and races with the new query | `flatMapLatest` |
| `combine(a, b, c)` with cold inputs that may never emit | Combined flow never produces — screen stuck on `initialValue` forever | Make every input a `StateFlow`, give cold inputs `onStart { emit(initial) }`, or sentinel them |
| `stateIn(...)` / `shareIn(...)` started from a composition-scoped scope (`rememberCoroutineScope()`, `LaunchedEffect`-launched) | Pipeline torn down on disposal, restarted on every recomposition cycle, no replay across config change | Move to a presenter / ViewModel scope; UI just collects |
| `flatMap` (= `flatMapConcat`) on a stream that should cancel previous work | Inner flows queue up serially; if the inner flow never completes, you deadlock | `flatMapLatest` for cancel-on-new, `flatMapMerge` only for true fan-out |
| `MutableSharedFlow<UiEvent>` for a result that must not be lost | UI in STOPPED misses the emission; replay default is 0 | Model as state with an `onConsumed` callback (see "Prefer Durable State" above) |
| `SharedFlow<Unit>` as a "trigger to refetch" | The trigger races subscribers; first emission can land before the collector is ready | Either use `MutableStateFlow<Long>` ticks (acknowledgement model) or give the work an explicit suspending entry point on the ViewModel |
| `combine` / `merge` / `flatMapLatest` constructed in the composable body | Pipeline rebuilt every recomposition — and lives in the wrong layer | Move to presenter; UI collects the resulting `StateFlow` |
| Bare `Channel` / `MutableStateFlow` / `MutableSharedFlow` in a public ViewModel API | Callers can mutate or close it; type-safety lost | Expose `Flow` / `StateFlow` / `SharedFlow` via `receiveAsFlow()` / `asStateFlow()` / `asSharedFlow()` |

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
- **`launchIn` on a UI-tied scope.** `launchIn(scope)` is hard to read and easy to leak. Inside a composable, prefer a correctly keyed `LaunchedEffect(flow) { flow.collect { ... } }` for ephemeral events. Inside a ViewModel, write `viewModelScope.launch { flow.collect { ... } }` explicitly.
- **Constructing the data pipeline in the composable.** `combine(...)`, `flatMapLatest { ... }`, or `stateIn(...)` written inside an `@Composable` body. Move the shaping to a presenter / state holder / ViewModel and let the UI consume one coherent `StateFlow<UiState>`.
- **`stateIn(rememberCoroutineScope(), ...)`.** Pipeline scoped to composition disposes when the screen leaves and rebuilds on the next entry. Almost always wrong — `stateIn` belongs in a `viewModelScope` (or equivalent long-lived scope) so the value survives configuration changes.
- **Modeling must-deliver outcomes as `MutableSharedFlow<UiEvent>`.** Replay-0 SharedFlows drop emissions when there is no active collector; if the UI is in STOPPED when the result arrives, it is gone forever. Move the outcome into state, with the UI calling back to clear it.
- **`zip` for screen state.** `zip` is pair-wise and waits for the slower side. For "latest of each input" use `combine`; if any input is cold, give it an initial value so `combine` actually emits.

## Primary Sources

- `https://kotlinlang.org/docs/flow.html`
- `https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/`
- `https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-shared-flow/`
- `https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/state-in.html`
- `https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/share-in.html`
- `https://developer.android.com/kotlin/flow`
- `https://developer.android.com/topic/architecture/ui-layer/state-production`
- `https://elizarov.medium.com/cold-flows-hot-channels-d74769805f9` (Roman Elizarov — cold flows, hot channels)
