# State

Compose state rules LLMs routinely get wrong. Most mistakes come from one of three places: state placed at the wrong altitude, `remember` without the right key, or treating `State<T>` like a value instead of an observable.

## State Hoisting — Where State Lives

Default shape for a screen:

```kotlin
// Stateful wrapper — owns the state, plugs into the ViewModel
@Composable
fun ProfileRoute(
    viewModel: ProfileViewModel = hiltViewModel(),
    onEditProfile: (userId: String) -> Unit,
) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    ProfileScreen(
        state = state,
        onRefresh = viewModel::refresh,
        onEditProfile = onEditProfile,
    )
}

// Stateless screen — pure function of its inputs
@Composable
fun ProfileScreen(
    state: ProfileState,
    onRefresh: () -> Unit,
    onEditProfile: (userId: String) -> Unit,
    modifier: Modifier = Modifier,
) {
    // ...
}
```

Rules:

- The **stateless** composable takes data and callbacks; it does not instantiate ViewModels, does not touch `remember { mutableStateOf(...) }` for screen state, and is easy to preview.
- The **stateful route** (sometimes called `*Route`, `*Host`, or `*Container`) owns the plumbing: ViewModel, lifecycle, navigation callbacks.
- Component-local UI state that is **not** worth hoisting can live inside the component — a text field's scroll position, a hover flag, an expanded/collapsed boolean for a card. Hoist only when a parent needs to observe or control it.

**LLM tell:** creating a ViewModel inside what should be the stateless screen, or using `remember { mutableStateOf(initialFromProps) }` — see the next section.

## `remember` Is Not An Initializer

The `remember` block runs once per composition, keyed on its `key` parameters. Forgetting the keys breaks state.

```kotlin
// Wrong — if `user` changes, the remembered value is stale
@Composable
fun Greeting(user: User) {
    val greeting = remember { "Hello, ${user.name}" }
    Text(greeting)
}

// Right — keyed on user
@Composable
fun Greeting(user: User) {
    val greeting = remember(user) { "Hello, ${user.name}" }
    Text(greeting)
}

// Also right — derived from input, no caching needed
@Composable
fun Greeting(user: User) {
    Text("Hello, ${user.name}")
}
```

Rules:

- If the remembered value is cheap to compute, do not `remember` it at all. `remember` exists for **expensive** computations and for values that need identity stability across recompositions (like a `CoroutineScope` or an `Animatable`).
- If the remembered value depends on inputs, pass them as keys: `remember(input1, input2) { ... }`.
- Never `remember { mutableStateOf(prop) }` to seed mutable state from a parameter. The seeded state never updates when `prop` changes. If the caller must be able to change it, hoist; if truly internal, use a proper derivation.

### The Keying Rule

**Any changing value read inside `remember`, `LaunchedEffect`, `DisposableEffect`, `produceState`, or the setup lambda of a remembered `derivedStateOf` must be one of:**

1. **Listed in the key arguments** so the cached value or in-flight effect restarts when the input changes;
2. **Intentionally fixed for the call-site lifetime** — a constant, a stable object the call-site owns and never replaces (a `CoroutineScope` from `rememberCoroutineScope()`, an `Animatable` from `remember { Animatable(...) }`, a state holder), or a value where capturing the *initial* value is the explicit goal; or
3. **Read through `rememberUpdatedState(value)`** when the effect should keep running across changes but invoke the latest version of the value (typically a callback).

If none of these apply, the value is captured stale and the bug is silent — the cache returns yesterday's result, the effect calls a closed-over old callback, the producer keeps running against the wrong inputs.

```kotlin
// Wrong — `userId` changes but the effect captures the original value forever
LaunchedEffect(Unit) {
    repository.subscribe(userId).collect { ... }
}

// Right — restart the effect on userId change
LaunchedEffect(userId) {
    repository.subscribe(userId).collect { ... }
}

// Right — keep the effect running, but always call the latest onTick
val currentOnTick by rememberUpdatedState(onTick)
LaunchedEffect(period) {
    while (isActive) { delay(period); currentOnTick() }
}

// Right — initial-value capture is the goal
val firstSeenAt = remember { Clock.System.now() }
```

**Valid exceptions, called out explicitly:**

- **Constants and compile-time literals.** `LaunchedEffect(Unit)`, `LaunchedEffect(true)` for genuine "run once on enter" work.
- **Stable objects owned by the call-site.** A `CoroutineScope`, `Animatable`, `LazyListState`, or state holder created by the same composable's own `remember` / `rememberXxxState`. Their identity does not change for the call site's lifetime, so they do not need keying — but if the composable can be re-keyed by a parent (`key(id) { ... }`), even those reset.
- **Initial-value-only captures.** When the *initial* value is the entire intent (a session-start timestamp, a screen-entry id), `remember { compute(input) }` without keying `input` is correct — but leave a `// initial-only` comment, otherwise it reads like a bug.
- **Callbacks read through `rememberUpdatedState`.** Use this when restarting the effect on every callback change would be wrong (a long-lived timer, a listener you do not want to re-register).

## `remember` vs `rememberSaveable`

- `remember` — survives recomposition. Does **not** survive configuration change or process death.
- `rememberSaveable` — survives both, using the `SavedStateRegistry`. The value must be `Parcelable`, `Serializable`, a primitive, or have a custom `Saver`.

Defaults to reach for:

- UI input state (text field content before submit, scroll position that does not come from `LazyListState`, page index) → `rememberSaveable`.
- Animation objects, coroutine scopes, focus requesters → `remember`.
- Data fetched from a repository, state computed from domain input → **neither** — keep it in the ViewModel and collect it.

## `derivedStateOf` Is For Reducing Recomposition

Use `derivedStateOf` **only** when:

1. You read **multiple** observable states, and
2. The combined output changes less frequently than the inputs, and
3. Downstream composables would otherwise recompose too often.

```kotlin
// Right — list scroll position changes continuously, but we only care about one boolean
val showScrollToTop by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 5 }
}
```

**LLM tell:** wrapping a single input in `derivedStateOf`.

```kotlin
// Wrong — no combining logic, no filtering
val name by remember(user) {
    derivedStateOf { user.name }
}

// Right — just read it
val name = user.name
```

### `derivedStateOf` And The Surrounding `remember`

`derivedStateOf` tracks **Compose `State<T>` reads inside its lambda**. It does **not** track plain values captured by the lambda. Those plain values are captured once when the surrounding `remember { ... }` runs.

```kotlin
// Wrong — `threshold` is a plain Int captured by the lambda. If the caller
// changes threshold, the derived state keeps using the original value.
@Composable
fun ScrollFlag(listState: LazyListState, threshold: Int) {
    val showFlag by remember {
        derivedStateOf { listState.firstVisibleItemIndex > threshold }
    }
}

// Right — key the surrounding remember on `threshold` so the derivedStateOf
// is rebuilt when the captured non-State value changes.
@Composable
fun ScrollFlag(listState: LazyListState, threshold: Int) {
    val showFlag by remember(threshold) {
        derivedStateOf { listState.firstVisibleItemIndex > threshold }
    }
}
```

The same rule from the keying section applies here: if the lambda captures a value that is not a tracked `State<T>`, that value either belongs in the `remember(...)` keys or is a constant / call-site-owned object whose identity is fixed for the call's lifetime.

## Observable vs Non-Observable Mutable State

Composables read observable state: `State<T>`, `MutableState<T>`, `SnapshotStateList`, `SnapshotStateMap`, and Flow via `collectAsStateWithLifecycle()`. Mutations to plain fields do not recompose.

```kotlin
// Wrong — plain var, recompositions will not see updates
class CounterHolder {
    var count = 0
}
// Right — observable
class CounterHolder {
    var count by mutableIntStateOf(0)
}
```

Use the **typed** factories for primitives:

- `mutableIntStateOf` / `mutableLongStateOf` / `mutableFloatStateOf` / `mutableDoubleStateOf`.
- Plain `mutableStateOf<Int>(0)` works but boxes the primitive into `Integer`, costing allocation and defeating the typed state machinery.

## State Holders Other Than ViewModel

For component-level state that is too big for inline `remember` but does not deserve a ViewModel, write a **state holder class**:

```kotlin
@Stable
class AppBarState(
    private val scope: CoroutineScope,
    private val listState: LazyListState,
) {
    var expanded by mutableStateOf(true)
        private set

    fun onScroll(delta: Float) {
        // collapse/expand logic driven by scroll delta
    }
}

@Composable
fun rememberAppBarState(listState: LazyListState): AppBarState {
    val scope = rememberCoroutineScope()
    return remember(listState, scope) { AppBarState(scope, listState) }
}
```

Guidelines:

- Mark the class `@Stable` and expose its mutable fields through a `State<T>` so Compose can track reads.
- Pair every state-holder class with a `rememberXxxState()` factory that keys on the constructor inputs.
- Do not expose `MutableState<T>` in the public API — expose `State<T>` or a plain property backed by `mutableStateOf`. See `references/component-api.md`.

## ViewModel Placement

Use `viewModel()` (or `hiltViewModel()`, `koinViewModel()`) **only at the route composable** — the entry point of a navigation destination. Passing a ViewModel deeper into the tree couples child composables to the DI graph and makes them unpreviewable.

```kotlin
// Wrong — child knows about ViewModel
@Composable
fun ProfileContent(viewModel: ProfileViewModel) { ... }

// Right — child takes data + callbacks
@Composable
fun ProfileContent(state: ProfileState, onAction: (ProfileAction) -> Unit) { ... }
```

**LLM tell:** `@Composable fun MyScreen(viewModel: MyViewModel = viewModel())` used from the middle of the tree. Keep the default only on the route, or drop the default entirely.

## Snapshot Lists And Maps

For collections that are **mutated in place** in the UI:

- `mutableStateListOf<T>()` / `mutableStateMapOf<K, V>()` from `androidx.compose.runtime`.
- These are observable. Mutations trigger recomposition of the specific readers.

If your state holder already exposes an immutable `List<T>` and replaces the whole list on update (`StateFlow<List<T>>`, `MutableState<List<T>>`, reducer-style state objects), that is also a valid shape. The real bug is hiding a mutable `ArrayList` / `mutableListOf(...)` inside state and then mutating it without changing the observed container — Compose will not see those internal mutations.

So:

- Use snapshot collections when the UI needs in-place mutation.
- Use immutable list replacement when your architecture already models state as whole-value updates.
- Avoid `mutableStateOf(mutableListOf(...))` unless you always replace the outer state after every change.

## Primary Sources

- `https://developer.android.com/develop/ui/compose/state`
- `https://developer.android.com/develop/ui/compose/state-hoisting`
- `https://developer.android.com/develop/ui/compose/architecture`
- `https://developer.android.com/topic/architecture/ui-layer/state-production`
- `https://developer.android.com/develop/ui/compose/state-saving`
