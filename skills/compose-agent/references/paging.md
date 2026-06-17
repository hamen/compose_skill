# Paging

Paging 3 in Compose is not "a lazy list that loads more." It is a **windowed, append-only stream** with its own load states, refresh semantics, and key stability rules. LLMs treat `LazyPagingItems` like `List<Item>` and reproduce the same three production bugs: **missing keys**, **ignored `LoadState`**, and **refresh wired from composition**.

Paging defects are usually lazy-list defects wearing a paginator costume. Cross-reference `performance.md` (keys, scroll cost), `state.md` (single source of truth for load UI), `concurrency.md` (lifecycle collection), and `effects.md` (when refresh belongs in an effect vs a click handler).

For a numeric audit of an existing codebase, use the sibling `jetpack-compose-audit` skill — paging smells score under Performance and State, not a separate category.

## When Paging vs a Normal List

| Situation | Use |
|-----------|-----|
| Bounded, in-memory collection (settings rows, tabs, static menu) | `LazyColumn` + `items(list)` |
| Network/DB-backed stream that grows as the user scrolls | `Pager` → `PagingData` → `collectAsLazyPagingItems()` |
| User expects pull-to-refresh over a paged feed | Paging + explicit refresh UI driven by `LoadState` |
| Small list loaded once from the ViewModel (`StateFlow<List<T>>`) | Plain list — do not add Paging ceremony |

**LLM tell:** introducing `Pager`, `PagingSource`, and `LazyPagingItems` for a screen that already holds a complete `List` in a `StateFlow`. Paging earns its complexity only when the dataset is **too large or too slow** to hold entirely in memory.

<https://developer.android.com/topic/libraries/architecture/paging/v3-compose>

## Collect Once, Lifecycle-Aware

`LazyPagingItems` comes from collecting a `Flow<PagingData<T>>`. In UI:

```kotlin
@Composable
fun FeedScreen(viewModel: FeedViewModel) {
    val lazyPagingItems = viewModel.feed.collectAsLazyPagingItems()
    FeedList(lazyPagingItems)
}
```

Guardrails:

- **`PagingData` is collected in the ViewModel** with `cachedIn(viewModelScope)` (or equivalent scope). Do not rebuild `Pager`/`PagingData` on every UI recomposition.
- Prefer **`collectAsLazyPagingItems()`** in the composable that renders the list. It is the supported Compose integration API.
- If the surrounding screen already uses lifecycle-aware Flow collection elsewhere, stay consistent — do not mix `collectAsState()` for paging with `collectAsStateWithLifecycle()` for other streams on the same screen without a reason.

**LLM tell:** `viewModel.feed.collectAsState()` then manually iterating — there is no supported `PagingData` → `List` shortcut for infinite feeds.

<https://developer.android.com/topic/libraries/architecture/paging/v3-compose#collect-lazyPagingItems>

## Render With `itemCount` and Stable Keys

The supported lazy integration uses **`items(count = lazyPagingItems.itemCount, key = …)`** (or the equivalent `items` overload that accepts `LazyPagingItems`). Keys must come from **stable domain identity**, not list index.

```kotlin
// LLM-generated — index keys break on prepend/refresh/reorder
LazyColumn {
    items(count = lazyPagingItems.itemCount, key = { index -> index }) { index ->
        val item = lazyPagingItems[index]
        if (item != null) FeedRow(item)
    }
}

// Correct — stable id; null slot = placeholder while page loads
LazyColumn {
    items(
        count = lazyPagingItems.itemCount,
        key = lazyPagingItems.itemKey { it.id },
    ) { index ->
        lazyPagingItems[index]?.let { FeedRow(it) }
    }
}
```

Rules:

- Use **`itemKey { it.<stableId> }`** when the item type is non-null in the lambda.
- Never use **`hashCode()`** on a non-`data class` or on objects that can collide across pages.
- If the backend can return **duplicate IDs** (merged feeds, reconnect storms), dedupe in the repository layer or synthesize keys (`"${source}-${id}"`) — Compose will crash with `Key ... was already used` otherwise.
- Do not call **`indexOf` / `indexOfFirst`** inside the item factory to recover identity — same O(n²) and crash profile as non-paging lazy lists. See `performance.md`.

**LLM tell:** `lazyPagingItems.itemSnapshotList.items.forEach { … }` or building a `List` from the snapshot and passing it to a child `LazyColumn`. Stay in the **`items(count = …)`** integration; the snapshot is for debugging/tests, not primary UI wiring.

<https://developer.android.com/topic/libraries/architecture/paging/v3-compose#display-lazylist>

## LoadState — Loading, Empty, Error, Append

`LazyPagingItems` exposes **`loadState`** (refresh / prepend / append). UI must branch on it — do not assume `itemCount > 0` means success.

| Signal | Typical UI |
|--------|------------|
| `loadState.refresh is LoadState.Loading` && `itemCount == 0` | Full-screen loading |
| `loadState.refresh is LoadState.Error` | Error + retry (`lazyPagingItems.retry()`) |
| `loadState.append is LoadState.Loading` | Footer spinner / item placeholder |
| `loadState.append is LoadState.Error` | Snackbar or inline "tap to retry" on footer |
| `loadState.refresh is LoadState.NotLoading` && `itemCount == 0` | Empty state |

```kotlin
when {
    lazyPagingItems.loadState.refresh is LoadState.Loading && lazyPagingItems.itemCount == 0 -> {
        LoadingContent()
    }
    lazyPagingItems.loadState.refresh is LoadState.Error -> {
        ErrorContent(onRetry = { lazyPagingItems.retry() })
    }
    else -> {
        FeedList(lazyPagingItems)
    }
}
```

**LLM tell:** rendering `LazyColumn` with zero items and no loading/error branch — users see a blank screen while refresh runs or after a failed fetch.

**LLM tell:** calling **`refresh()`** unconditionally in a `LaunchedEffect(Unit)` or in the composable body to "load data on enter." Initial load is **`PagingData`'s job**; `refresh()` is for **user-initiated** pull-to-refresh or explicit reload actions. One-shot navigation args belong in the ViewModel/`Pager` factory, not a composition-time refresh loop.

<https://developer.android.com/reference/kotlin/androidx/paging/compose/LazyPagingItems>

## Pull-to-Refresh and Retry

Wire refresh to **user events** or explicit reload actions:

```kotlin
PullToRefreshBox(
    isRefreshing = lazyPagingItems.loadState.refresh is LoadState.Loading,
    onRefresh = { lazyPagingItems.refresh() },
) {
    FeedList(lazyPagingItems)
}
```

Use **`retry()`** for error recovery after `LoadState.Error`. Do not spin a separate `mutableStateOf(isLoading)` that duplicates `loadState` unless the design needs decoupled UX — two sources of truth for loading is an LLM favorite and a bug farm.

## API Shape for Reusable Components

Shared list components should take **`LazyPagingItems<T>`** (or a narrow interface) plus callbacks — not a materialized `List<T>` that defeats paging.

For design-system wrappers:

- Accept `modifier: Modifier = Modifier` on the outermost scroll container.
- Do not expose **`MutableState`** for load state when `loadState` already exists on `LazyPagingItems`.
- Keep **`refresh` / `retry`** as callback parameters if the parent owns the `LazyPagingItems` instance.

See `component-api.md` for parameter order and slot conventions.

## Anti-Pattern Checklist

Before returning paging UI code, verify:

- [ ] Paging is justified (unbounded / paged data source), not a `StateFlow<List>` in disguise
- [ ] `PagingData` is **`cachedIn`** appropriate scope in the ViewModel
- [ ] List uses **`items(count = lazyPagingItems.itemCount, key = …)`** with **stable ids**
- [ ] No **index-only** or **`hashCode()`** keys on merge-prone feeds
- [ ] **`LoadState`** drives initial loading, empty, error, and append footer — not a parallel manual flag
- [ ] **`refresh()` / `retry()`** are user-driven or explicitly documented — not fired from composition on every recomposition
- [ ] No **`indexOf`** identity recovery inside item factories
- [ ] Placeholder slots (`lazyPagingItems[i] == null`) handled without crashing

## Primary Sources

- <https://developer.android.com/topic/libraries/architecture/paging/v3-compose>
- <https://developer.android.com/topic/libraries/architecture/paging/v3-overview>
- <https://developer.android.com/reference/kotlin/androidx/paging/compose/package-summary>
- <https://developer.android.com/develop/ui/compose/lists> — lazy list keys (applies to paging items)

When supplemental blog posts disagree with these, the Android Developers pages win.
