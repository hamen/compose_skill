# Paging

Paging 3 in Compose is not "a lazy list that loads more." It is a **windowed paged stream** (primarily append, with prepend and refresh generations) carrying load states, refresh semantics, and key rules of its own. LLMs treat `LazyPagingItems` like `List<Item>` and reproduce the same three production bugs:

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
| Error on first load (nothing on screen yet) | `loadState.refresh is LoadState.Error && itemCount == 0` + `retry()` |
| Refresh error with content already shown | transient message (snackbar) + keep the list — **do not** replace the feed with a full-screen error |
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
        // Full-screen error ONLY when there is nothing to show. A refresh that
        // fails while a feed is already on screen must NOT blow the list away.
        lazyPagingItems.loadState.refresh is LoadState.Error && lazyPagingItems.itemCount == 0 -> {
            ErrorContent(onRetry = { lazyPagingItems.retry() })
        }
        lazyPagingItems.loadState.refresh is LoadState.NotLoading && lazyPagingItems.itemCount == 0 -> {
            EmptyContent()
        }
        else -> {
            // itemCount > 0: keep the list. A failed pull-to-refresh surfaces as a
            // transient message, not a full-screen replacement.
            val refreshState = lazyPagingItems.loadState.refresh
            // Key on the Error itself (or a flag), not the whole loadState, so the
            // snackbar fires once per failure and does not re-show on unrelated state churn.
            LaunchedEffect(refreshState is LoadState.Error) {
                if (refreshState is LoadState.Error) {
                    // snackbarHostState.showSnackbar(...) with a Retry action -> lazyPagingItems.retry()
                }
            }
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
        when (lazyPagingItems.loadState.append) {
            is LoadState.Loading -> item { AppendSpinner() }
            is LoadState.Error -> item { AppendRetryRow(onRetry = { lazyPagingItems.retry() }) }
            else -> Unit
        }
    }
}
```

ViewModel guardrail: expose `Flow<PagingData<Post>>` built once and **`cachedIn(viewModelScope)`**. Do not rebuild `Pager` / `PagingData` on UI recomposition.

**LLM tell:** `viewModel.feed.collectAsState()` then manually iterating — there is no supported `PagingData` → `List` shortcut for infinite feeds.

**LLM tell:** `viewModel.feed.collectAsStateWithLifecycle()` on a `Flow<PagingData<T>>`. The lifecycle rule that holds for every other screen does **not** transfer here. `PagingData` is a one-shot stream consumed by a dedicated differ — there is no supported path to collect it through `collectAsStateWithLifecycle`; `collectAsLazyPagingItems()` is the collector that drives the differ. (Note it is not "lifecycle-aware" in the `repeatOnLifecycle` sense — it collects for as long as it is in composition; backgrounding does not pause it. That is expected for paging and not something to "fix" by reaching for `collectAsStateWithLifecycle`.) Treat "this Flow needs `collectAsStateWithLifecycle`" as not applying to `PagingData`.

<https://developer.android.com/topic/libraries/architecture/paging/v3-compose#collect-lazyPagingItems>

## Keys — Stable Domain Identity, Not Index

The supported integration is **`items(count = lazyPagingItems.itemCount, key = lazyPagingItems.itemKey { … })`**. The older `items(lazyPagingItems)` overload on `LazyListScope` is **deprecated** — use `itemKey` with the standard lazy `items(count, key, …)` API instead (`LazyColumn`, `LazyVerticalGrid`, and friends). For a `HorizontalPager`/`VerticalPager` driven by `LazyPagingItems`, the wiring differs — pass `pageCount = { lazyPagingItems.itemCount }` and read with `lazyPagingItems[page]`, supplying a stable `key` — so don't assume the `items(count, key)` shape transfers verbatim.

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
- For **heterogeneous** paged feeds (ads, headers, mixed row types), pass **`contentType = lazyPagingItems.itemContentType { … }`** alongside `itemKey`. Without it Compose cannot reuse slots across differently-shaped items and recomposition cost climbs — the same rule as `contentType` on a plain lazy list.

<https://developer.android.com/reference/kotlin/androidx/paging/compose/LazyPagingItems#itemKey(kotlin.Function1)>

## LoadState and Refresh

`LazyPagingItems.loadState` has **refresh**, **prepend**, and **append**. UI must branch — do not assume `itemCount > 0` means success.

| Signal | Typical UI |
|--------|------------|
| `refresh is Loading` && `itemCount == 0` | Full-screen loading |
| `refresh is Error` && `itemCount == 0` | Full-screen error + `retry()` |
| `refresh is Error` && `itemCount > 0` | Transient message (snackbar) — keep the list |
| `refresh is NotLoading` && `itemCount == 0` | Empty state |
| `append is Loading` | Footer spinner / row placeholder |
| `append is Error` | Snackbar or inline retry on footer |

**LLM tell:** rendering `LazyColumn` with zero items and no loading/error branch — users see a blank screen while refresh runs or after a failed fetch.

**LLM tell:** a bare `refresh is LoadState.Error -> ErrorContent()` branch with **no `itemCount` guard** — a transient pull-to-refresh failure then wipes an already-loaded feed and replaces it with a full-screen error. Gate the full-screen error on `itemCount == 0`; surface refresh errors that arrive with content already on screen as a snackbar/inline retry instead.

**LLM tell:** calling **`refresh()`** unconditionally in `LaunchedEffect(Unit)` or in the composable body to "load data on enter." Initial load is **`PagingData`'s job**; `refresh()` is for **user-initiated** reload. One-shot navigation args belong in the ViewModel / `Pager` factory, not a composition-time refresh loop.

**LLM tell:** a parallel `mutableStateOf(isLoading)` shadowing `loadState` — two sources of truth for the same UX. Prefer `loadState` unless the design genuinely decouples them.

Use **`retry()`** after `LoadState.Error`. Wire **`refresh()`** to pull-to-refresh or explicit reload actions only.

**DB-backed / `RemoteMediator` feeds:** keep branching on the **combined `loadState.refresh`** for normal screen loading — it already defers to the mediator and only reports `NotLoading` once **both** `source` and `mediator` have settled, so it will not blank the screen early. Reach for `loadState.source.refresh` / `loadState.mediator?.refresh` **only** when you deliberately render cached content while a network refresh continues in the background (explicit cache-vs-network UI). Branching on `source.refresh` for the default load is a tell — it shows an empty state while the mediator is still fetching. This reference is UI guidance, not a `RemoteMediator` cookbook.

**`prepend`** usually needs no UI of its own (bidirectional feeds are rare); when it does, mirror the `append` pattern — a header spinner on `prepend is Loading`, a header retry on `prepend is Error`. Do not invent a separate loading flag for it.

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

**Do not skip placeholders.** When `lazyPagingItems[index]` is `null`, render a placeholder — do not assume every slot is non-null. Never `!!` the slot: it NPE-crashes the first time a page is still loading.

Whether `null` slots occur at all is set by **`PagingConfig.enablePlaceholders`** (a repository/`Pager` decision, not a UI one). Placeholders **on** → `get(index)` can return `null` and you **must** handle it. Placeholders **off** → slots are never `null`, but the list jumps as pages arrive. Match the UI branch to the config: don't write a placeholder branch that can never run, and don't assume non-null when placeholders are enabled.

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
- `concurrency.md` — lifecycle-aware collection of **other** (non-`PagingData`) flows on the same screen; it does not apply to `LazyPagingItems`
- `effects.md` — refresh belongs in event handlers, not composition body

## Primary Sources

- <https://developer.android.com/topic/libraries/architecture/paging/v3-compose>
- <https://developer.android.com/topic/libraries/architecture/paging/v3-overview>
- <https://developer.android.com/reference/kotlin/androidx/paging/compose/package-summary>
- <https://developer.android.com/develop/ui/compose/lists> — lazy list keys (applies to paging items)

When supplemental blog posts disagree with these, the Android Developers pages win.
