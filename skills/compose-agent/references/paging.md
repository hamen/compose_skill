# Paging

Paging 3 in Compose is not "a lazy list that loads more." It is a **windowed, append-only stream** with load states, refresh semantics, and key rules of its own. LLMs treat `LazyPagingItems` like `List<Item>` and reproduce the same three production bugs:

1. **Missing or index-only keys** on a paginated stream
2. **Ignored `LoadState`** — blank screens, stuck spinners, silent errors
3. **`refresh()` wired from composition** instead of user actions

For a numeric audit of an existing codebase, use the sibling `jetpack-compose-audit` skill — paging smells score under Performance and State, not a separate category.

## Decision Table

| You are doing… | Use |
|---|---|
| Feed that grows as the user scrolls (network / DB window) | `Flow<PagingData<T>>` + `collectAsLazyPagingItems()` |
| List already complete in `StateFlow<List<T>>` | `LazyColumn` + `items(list)` — **do not add Paging** |
| First paint while refresh runs | branch on `loadState.refresh is LoadState.Loading && itemCount == 0` |
| Error on first load | `loadState.refresh is LoadState.Error` + `retry()` |
| Empty feed after successful load | `loadState.refresh is LoadState.NotLoading && itemCount == 0` |
| Footer "loading more" | `loadState.append is LoadState.Loading` |
| Footer error | `loadState.append is LoadState.Error` + retry affordance |
| User pull-to-refresh / explicit reload | `refresh()` on gesture — **not** in composition |
| Stable lazy identity | `items(count = …, key = lazyPagingItems.itemKey { it.id })` |
| `@Preview` of a paged screen | `flowOf(PagingData.from(samples)).collectAsLazyPagingItems()` |

**LLM tell:** introducing `Pager`, `PagingSource`, and `LazyPagingItems` for a screen that already holds a complete `List` in a `StateFlow`. Paging earns its complexity only when the dataset is **too large or too slow** to hold entirely in memory.

<https://developer.android.com/topic/libraries/architecture/paging/v3-compose>

## Golden Path

One screen-level recipe. Copy this shape; swap names.

```kotlin
@Composable
fun FeedScreen(viewModel: FeedViewModel) {
    val lazyPagingItems = viewModel.feed.collectAsLazyPagingItems()

    when {
        lazyPagingItems.loadState.refresh is LoadState.Loading && lazyPagingItems.itemCount == 0 -> {
            LoadingContent()
        }
        lazyPagingItems.loadState.refresh is LoadState.Error -> {
            ErrorContent(onRetry = { lazyPagingItems.retry() })
        }
        lazyPagingItems.loadState.refresh is LoadState.NotLoading && lazyPagingItems.itemCount == 0 -> {
            EmptyContent()
        }
        else -> {
            PullToRefreshBox(
                isRefreshing = lazyPagingItems.loadState.refresh is LoadState.Loading,
                onRefresh = { lazyPagingItems.refresh() },
            ) {
                FeedList(lazyPagingItems)
            }
        }
    }
}

@Composable
private fun FeedList(lazyPagingItems: LazyPagingItems<Post>) {
    LazyColumn {
        items(
            count = lazyPagingItems.itemCount,
            key = lazyPagingItems.itemKey { it.id },
        ) { index ->
            when (val post = lazyPagingItems[index]) {
                null -> PostPlaceholder()
                else -> PostRow(post)
            }
        }
        if (lazyPagingItems.loadState.append is LoadState.Loading) {
            item { AppendSpinner() }
        }
    }
}
```

ViewModel guardrail: expose `Flow<PagingData<Post>>` built once and **`cachedIn(viewModelScope)`**. Do not rebuild `Pager` / `PagingData` on UI recomposition.

**LLM tell:** `viewModel.feed.collectAsState()` then manually iterating — there is no supported `PagingData` → `List` shortcut for infinite feeds.

<https://developer.android.com/topic/libraries/architecture/paging/v3-compose#collect-lazyPagingItems>

## Keys — Stable Domain Identity, Not Index

The supported integration is **`items(count = lazyPagingItems.itemCount, key = lazyPagingItems.itemKey { … })`**. The older `items(lazyPagingItems)` overload on `LazyListScope` is **deprecated** — use `itemKey` with the standard lazy `items(count, key, …)` API instead (works for `LazyColumn`, `LazyVerticalGrid`, `HorizontalPager`, etc.).

```kotlin
// LLM-generated — index keys break on prepend / refresh / reorder
items(count = lazyPagingItems.itemCount, key = { index -> index }) { index -> … }

// Correct — stable id; null slot = placeholder while a page loads
items(
    count = lazyPagingItems.itemCount,
    key = lazyPagingItems.itemKey { it.id },
) { index ->
    lazyPagingItems[index]?.let { PostRow(it) } ?: PostPlaceholder()
}
```

Rules:

- Keys come from **stable domain ids**, never bare index on paginated feeds.
- Never use **`hashCode()`** on a non-`data class` or on objects that can collide across pages.
- Duplicate backend IDs (merged feeds, reconnect storms) → dedupe in the repository or synthesize keys (`"${source}-${id}"`). Compose crashes with `Key ... was already used` otherwise.
- No **`indexOf` / `indexOfFirst`** inside the item factory — same O(n²) and crash profile as non-paging lazy lists. See `performance.md`.

<https://developer.android.com/reference/kotlin/androidx/paging/compose/LazyPagingItems#itemKey(kotlin.Function1)>

## LoadState and Refresh

`LazyPagingItems.loadState` has **refresh**, **prepend**, and **append**. UI must branch — do not assume `itemCount > 0` means success.

| Signal | Typical UI |
|--------|------------|
| `refresh is Loading` && `itemCount == 0` | Full-screen loading |
| `refresh is Error` | Error + `retry()` |
| `refresh is NotLoading` && `itemCount == 0` | Empty state |
| `append is Loading` | Footer spinner / row placeholder |
| `append is Error` | Snackbar or inline retry on footer |

**LLM tell:** rendering `LazyColumn` with zero items and no loading/error branch — users see a blank screen while refresh runs or after a failed fetch.

**LLM tell:** calling **`refresh()`** unconditionally in `LaunchedEffect(Unit)` or in the composable body to "load data on enter." Initial load is **`PagingData`'s job**; `refresh()` is for **user-initiated** reload. One-shot navigation args belong in the ViewModel / `Pager` factory, not a composition-time refresh loop.

**LLM tell:** a parallel `mutableStateOf(isLoading)` shadowing `loadState` — two sources of truth for the same UX. Prefer `loadState` unless the design genuinely decouples them.

Use **`retry()`** after `LoadState.Error`. Wire **`refresh()`** to pull-to-refresh or explicit reload actions only.

<https://developer.android.com/reference/kotlin/androidx/paging/compose/LazyPagingItems>

## Hard Nos

**Do not materialize the window in composition.**

```kotlin
// LLM tell — snapshot is for debug/tests, not primary UI
lazyPagingItems.itemSnapshotList.items.forEach { … }
val filtered = remember(query) {
    lazyPagingItems.itemSnapshotList.filter { it.title.contains(query) }
}
```

Filtering, sorting, and search belong in the **ViewModel / PagingSource / `flatMapLatest` on the `Flow<PagingData<T>>`**, not in the composable. Materializing defeats the lazy window and reintroduces memory churn.

**Do not skip placeholders.** When `lazyPagingItems[index]` is `null`, render a placeholder — do not assume every slot is non-null.

## Preview

```kotlin
@Preview
@Composable
private fun FeedListPreview() {
    val lazyPagingItems = flowOf(PagingData.from(samplePosts)).collectAsLazyPagingItems()
    FeedList(lazyPagingItems)
}
```

**LLM tell:** faking a paged screen with `List<Post>` passed to a non-paging `LazyColumn` — previews drift from production wiring. Use `PagingData.from(...)` for Compose previews.

## Anti-Pattern Checklist

Before returning paging UI code, verify:

- [ ] Paging is justified — not a `StateFlow<List>` in disguise
- [ ] `PagingData` is **`cachedIn`** in the ViewModel
- [ ] **`items(count, key = itemKey { stableId })`** — no index-only keys on paginated feeds
- [ ] **`LoadState`** drives loading / empty / error / append — no duplicate manual loading flag
- [ ] **`refresh()` / `retry()`** are user-driven — not fired from composition each recomposition
- [ ] No **`itemSnapshotList`** primary UI path; no filter/sort in the composable
- [ ] No **`indexOf`** identity recovery inside item factories
- [ ] **`null` slots** render placeholders
- [ ] Reusable list APIs take **`LazyPagingItems<T>`** (or callbacks from the owner) — not a materialized `List<T>`; see `component-api.md` for `modifier` and parameter order

## Cross-References

- `performance.md` — lazy keys, scroll cost, duplicate-key crashes
- `state.md` — single source of truth for load UI
- `concurrency.md` — lifecycle-aware collection on the same screen
- `effects.md` — refresh belongs in event handlers, not composition body

## Primary Sources

- <https://developer.android.com/topic/libraries/architecture/paging/v3-compose>
- <https://developer.android.com/topic/libraries/architecture/paging/v3-overview>
- <https://developer.android.com/reference/kotlin/androidx/paging/compose/package-summary>
- <https://developer.android.com/develop/ui/compose/lists> — lazy list keys (applies to paging items)

When supplemental blog posts disagree with these, the Android Developers pages win.
