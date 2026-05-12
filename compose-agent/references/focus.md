# Compose Focus And Keyboard Navigation

Focus is stateful UI behavior. Make focus targets explicit, request focus after composition succeeds, preserve focus by semantic identity, and test with the same input model users use: keyboard, D-pad, remote, or accessibility focus.

Use this reference when UI:

- runs on TV, desktop, ChromeOS, keyboard-first Android, or remote-control devices
- uses `FocusRequester`, `focusRequester`, `focusProperties`, `focusable`, `onFocusChanged`, or key handlers
- needs initial focus, restored focus, directional navigation, back / escape handling, modal focus traps, or focused-item tests

## Build Focus Targets Deliberately

Start with components that are already focusable, then add only the hooks the behavior needs.

| Need | Add |
|---|---|
| Normal `Button`, `TextField`, clickable/selectable behavior | Nothing extra |
| Programmatic initial or restored focus | `FocusRequester` + `Modifier.focusRequester(...)` |
| UI reacts to focus state | `Modifier.onFocusChanged { ... }` |
| Custom interactive surface | `Modifier.focusable()` plus role/semantics as appropriate |

Avoid adding `FocusRequester` and `onFocusChanged` to every button. Use them only where focus ownership or focus state matters.

## Request Focus From An Effect

Never call `requestFocus()` in the composable body.

```kotlin
val firstField = remember { FocusRequester() }

TextField(
    value = name,
    onValueChange = onNameChange,
    modifier = Modifier.focusRequester(firstField),
)

LaunchedEffect(firstField) {
    firstField.requestFocus()
}
```

If the target appears after loading, key the request to that condition:

```kotlin
LaunchedEffect(items.isNotEmpty()) {
    if (items.isNotEmpty()) {
        firstItemRequester.requestFocus()
    }
}
```

For lazy content, request focus only after the target item is composed. Store requesters by stable item id when lists can reorder.

## Directional Navigation

Use natural spatial search first. Add `focusProperties` only when the default graph is wrong:

```kotlin
Modifier.focusProperties {
    up = headerRequester
    down = firstRowRequester
    left = FocusRequester.Cancel
}
```

Too many hard-coded links make the focus graph stale when layout changes. Override broken edges, traps, and required jumps; let default search handle normal grids and rows.

## Key Events

Use key handlers for behavior that is not normal click or traversal:

```kotlin
Modifier.onPreviewKeyEvent { event ->
    if (event.type == KeyEventType.KeyUp && event.key == Key.Back) {
        onBack()
        true
    } else {
        false
    }
}
```

Return `true` only for keys you actually consume. Broad consumption breaks text entry, accessibility shortcuts, and parent navigation.

For rapid D-pad input, throttle or coalesce at the owner of the expensive behavior, such as row paging or scroll, not across the whole screen.

## Focus Restoration

Preserve focus by semantic identity:

- track selected or focused item id, not only index
- use stable lazy-list `key =` values
- after refresh, re-request focus for the same id if it still exists
- if it no longer exists, choose a deterministic fallback: nearest neighbor, first item, or parent container

Do not infer focus from selection unless the product intentionally treats them as the same concept.

## Testing

Drive focus through user input and assert focused semantics:

```kotlin
composeTestRule.onNodeWithTag("screen").performKeyInput {
    pressKey(Key.DirectionDown)
}

composeTestRule.onNodeWithTag("play-button").assertIsFocused()
```

Use screenshot tests for focus ring appearance. Use semantics tests for deterministic focus ownership.

## Common Mistakes

| Mistake | Fix |
|---|---|
| `requestFocus()` in the composable body | Move to `LaunchedEffect` |
| Initial focus keyed to `Unit` while target appears later | Key to loaded/visible condition |
| Requesters stored by lazy-list index | Store by stable item id |
| Everything gets custom `focusProperties` | Override only broken edges |
| Key handler returns `true` for all keys | Consume only handled keys |
| TV/keyboard UI tested with clicks only | Send key input and assert focus |

## Primary Sources

- `https://developer.android.com/develop/ui/compose/touch-input/focus`
- `https://developer.android.com/develop/ui/compose/touch-input/keyboard-input`
- `https://developer.android.com/develop/ui/compose/testing`
- `https://developer.android.com/develop/ui/compose/accessibility`
