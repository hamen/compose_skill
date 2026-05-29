# Concurrency, Flows, And Lifecycle

The Coroutines / Flow model is Android's first-class async primitive. Compose sits on top of it. This file covers what the LLM keeps getting wrong at the boundary.

## Collecting Flows In UI

**Always use `collectAsStateWithLifecycle()`** for Flows that drive UI.

```kotlin
// Wrong
val user by viewModel.user.collectAsState()

// Right
val user by viewModel.user.collectAsStateWithLifecycle()
```

Why: `collectAsState()` keeps collecting while the composable is in the composition tree, which includes the back stack and paused screens. For a `StateFlow` backed by a cold upstream (Room, network, sensor), that means the upstream keeps running and burns battery.

`collectAsStateWithLifecycle()` respects `Lifecycle.State.STARTED` by default. Rare reason to change it: if you need the flow to keep emitting while the app is backgrounded (e.g. a media player service wrapper). Then pass `minActiveState = Lifecycle.State.CREATED` with intent.

Requires `implementation("androidx.lifecycle:lifecycle-runtime-compose:<version>")`.

### Initial Values

For a `StateFlow<T>`:

```kotlin
val state by viewModel.state.collectAsStateWithLifecycle()
```

The overload without `initialValue` reuses `StateFlow.value` — no dummy initial, no extra recomposition.

For a `Flow<T>` that is not a `StateFlow`:

```kotlin
val events by viewModel.events.collectAsStateWithLifecycle(initialValue = emptyList())
```

Pick an `initialValue` that matches the empty / loading semantics of the screen — do not default to `null` and then `?.let { ... }` everywhere.

## `repeatOnLifecycle` — The Non-Compose Counterpart

When collecting in a non-composable coroutine (e.g. a `LifecycleOwner`'s lifecycle scope, an Activity), the lifecycle-aware primitive is `repeatOnLifecycle`:

```kotlin
lifecycleScope.launch {
    lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.state.collect { render(it) }
    }
}
```

Do not roll your own with `launchWhenStarted { ... }`; that API is deprecated because it silently holds the coroutine in suspend and can lead to resource leaks.

## ViewModel Scope — Where Async Work Lives

- UI-triggered work (loading a profile on screen entry, posting a form, refreshing on pull): belongs in the **ViewModel**, launched with `viewModelScope`.
- UI-local work that is tied to composition (animations, scroll-to calls): belongs in composition with `LaunchedEffect` or `rememberCoroutineScope`.

```kotlin
// ViewModel
class ProfileViewModel(private val repo: ProfileRepository) : ViewModel() {
    private val _state = MutableStateFlow<ProfileState>(ProfileState.Loading)
    val state: StateFlow<ProfileState> = _state.asStateFlow()

    fun load(userId: String) = viewModelScope.launch {
        _state.value = ProfileState.Loading
        _state.value = runCatching { repo.load(userId) }
            .fold(::ProfileState.Loaded, ::ProfileState.Error)
    }
}
```

Guardrails:

- `viewModelScope` cancels on `ViewModel.onCleared()`. It is the correct scope for anything whose result should be visible to the UI layer.
- Never `GlobalScope.launch` from a ViewModel or a composable.
- Do not expose `MutableStateFlow` publicly. Expose `StateFlow` via `asStateFlow()`.
- Do not expose `Channel` publicly. Expose `Flow` via `receiveAsFlow()` / `consumeAsFlow()`.

## One-Shot Events From ViewModel To UI

Do not model events as `StateFlow<Event?>` with manual nulling — races are easy, and re-emission on configuration change is awkward. First decide whether the event must survive lifecycle gaps:

- If the UI must eventually render or acknowledge it (snackbar message, validation result, navigation decision), model it as UI state and clear it after the UI reports that it was handled. This matches Android's UI events guidance.
- If it is an ephemeral, single-collector signal where loss during cancellation is acceptable, use a `Channel<Event>` plus `receiveAsFlow()`.
- If multiple collectors must observe the same signal, use `MutableSharedFlow` / `SharedFlow` and configure replay/buffering deliberately.

Ephemeral `Channel` shape:

```kotlin
// ViewModel
private val _events = Channel<UiEvent>(Channel.BUFFERED)
val events: Flow<UiEvent> = _events.receiveAsFlow()

fun onSave() = viewModelScope.launch {
    when (val result = repo.save()) {
        is Success -> _events.send(UiEvent.NavigateBack)
        is Failure -> _events.send(UiEvent.ShowError(result.message))
    }
}

// UI
LaunchedEffect(Unit) {
    viewModel.events.collect { event ->
        when (event) {
            UiEvent.NavigateBack -> onBack()
            is UiEvent.ShowError -> snackbar.showSnackbar(event.message)
        }
    }
}
```

Channels are hot and buffered — events sent while no collector is attached are buffered up to `Channel.BUFFERED` slots by default. They are still not durable state: `receiveAsFlow()` fans out to collectors, and an element can be lost if a collector is cancelled after receiving it but before processing it. Use state for must-deliver UI outcomes.

## Flow Transformations

For deeper Flow operator selection (cold vs hot, `flatMap` variants, `combine`/`merge`/`zip`, error handling, backpressure, `asStateFlow()` exposure), see `flows.md`. The notes here are the minimum a lifecycle-focused review needs.

- Prefer `stateIn(...)` when converting a cold flow into a `StateFlow` for the ViewModel's public API, with `SharingStarted.WhileSubscribed(5_000)` as the default start strategy — keeps the upstream alive for 5 seconds across config changes but tears it down when nobody is watching.
- Use `flatMapLatest` for search-style queries where a new input invalidates the previous flow.
- Use `distinctUntilChanged()` on flows backed by noisy sources (sensors, scroll positions) before driving UI off them.
- Do not chain `.flowOn(Dispatchers.IO)` at the end of a UI flow — dispatcher switching near the UI is usually wrong. Apply `.flowOn(...)` adjacent to the source that actually needs it.

## Dispatchers

- UI reads + state mutations from the ViewModel run on the **main dispatcher**. Compose state and `StateFlow.value = ...` do not need explicit dispatcher switching.
- I/O (Room, Retrofit, file) has its own machinery — Room generates suspend functions that already switch dispatchers; Retrofit with suspend returns on the main thread but the network call runs off it. No explicit `withContext(Dispatchers.IO)` needed unless you are calling a blocking Java API.
- CPU-bound work (JSON parsing by hand, image decoding): `withContext(Dispatchers.Default)`.
- Do **not** use `Dispatchers.Main.immediate` to force ordering — the need for it usually means a state mutation is happening in the wrong place.

## Cancellation

- Coroutine cancellation is cooperative. Long loops should check `isActive` or use `yield()`. Blocking JVM calls do not respect cancellation — wrap them in `withContext(Dispatchers.IO) { runInterruptible { ... } }` if you need cancel-by-interrupt.
- `viewModelScope.launch { ... }` blocks cancel automatically when the ViewModel is cleared. Do not also wrap them in `try/finally` cleanup unless you have state to release.
- `LaunchedEffect` blocks cancel automatically when the composable leaves composition or when keys change. You do not need `try/finally { cancel() }`.

## Common LLM Mistakes

- **`runBlocking`** in UI code. Always wrong.
- **`Dispatchers.IO` inside composables.** Composables should not switch dispatchers at all. Move the work to the ViewModel.
- **`MutableStateFlow(...)` field on the ViewModel exposed publicly.** Expose `StateFlow` via `asStateFlow()`.
- **Collecting a flow by launching inside a composable body**: `scope.launch { flow.collect { state = it } }`. Use `collectAsStateWithLifecycle()`.
- **Leaking `ViewModel` from a child composable into a `by viewModels()` call.** ViewModel instantiation belongs on the route composable only (see `references/state.md`).
- **Choosing `_state.emit(...)` vs `_state.value = ...` for the wrong reason.** `MutableStateFlow.value` is thread-safe; it is not "main-thread-confined". Prefer `.value = ...` for straightforward synchronous state updates, `emit(...)` when you are implementing a suspending `FlowCollector` API, and `update { ... }` when the next value depends on the current one.
- **`launchIn` on the UI scope.** Works, but harder to read than `collect { ... }` inside a `LaunchedEffect`. Prefer the explicit form.

## Primary Sources

- `https://developer.android.com/topic/libraries/architecture/coroutines`
- `https://developer.android.com/topic/architecture/ui-layer/state-production`
- `https://developer.android.com/kotlin/coroutines/coroutines-best-practices`
- `https://kotlinlang.org/docs/flow.html`
- `https://developer.android.com/reference/kotlin/androidx/lifecycle/compose/package-summary` (`collectAsStateWithLifecycle`)
