# Compose UI Testing

Test the smallest UI contract that proves the behavior. Prefer plain state-driven Compose UI tests for rendering and callback wiring. Use integration tests only when lifecycle, navigation, dependency injection, or platform behavior is the thing being tested.

## Choose The Test Shape

| Need to prove | Test shape |
|---|---|
| Text, loading, empty, error, enabled / disabled branches | Plain Compose UI test |
| Clicks, text input, callbacks | Plain Compose UI test with captured callbacks |
| Focus, keyboard, D-pad, TV navigation | Compose UI test with key input; see `focus.md` |
| Layout, clipping, typography, elevation, colors, focus ring | Screenshot test |
| State holder transforms inputs into UI state | State holder unit test plus one wiring smoke test |
| Navigation, lifecycle, DI graph, platform permission behavior | Integration test |

## Prefer Plain UI Composables

If a screen has a state-holder/UI split, test the plain UI composable:

```kotlin
@Test
fun saveClickCallsCallback() {
    var saved = false

    composeTestRule.setContent {
        ProfileScreen(
            state = ProfileUiState(name = "Ada", canSave = true),
            onNameChange = {},
            onSaveClick = { saved = true },
            onBackClick = {},
        )
    }

    composeTestRule.onNodeWithText("Save").performClick()

    assertThat(saved).isTrue()
}
```

Do not construct a ViewModel, DI graph, repository, navigator, or Activity just to prove that a button is displayed or a callback fires.

## Semantics First

Use user-visible semantics before test tags:

- Text exists: `onNodeWithText(...)`.
- Icon-only action exists: content description.
- Button enabled / disabled: `assertIsEnabled()` / `assertIsNotEnabled()`.
- Selected, focused, checked, toggled: semantics assertions.
- Content absent: `assertDoesNotExist()`.

Use `testTag` for nodes with no stable visible semantics, duplicate labels, or layout/screenshot anchors. Do not tag everything by default.

## Callback And State Assertions

For plain callback values, assert directly after the UI action. Use `runOnIdle` only when Compose must finish applying snapshot state, recomposition, or queued UI work first.

```kotlin
var selectedId: String? = null

composeTestRule.setContent {
    MovieList(
        movies = listOf(MovieUi(id = "m1", title = "Arrival")),
        onMovieClick = { selectedId = it },
    )
}

composeTestRule.onNodeWithText("Arrival").performClick()

assertThat(selectedId).isEqualTo("m1")
```

## Screenshot Tests

Use screenshot tests for contracts semantics cannot prove:

- spacing, alignment, clipping, and density behavior
- typography, elevation, color, shadows, and theme variants
- image composition, gradients, overlays, and loading skeletons
- visual focus treatment

Keep screenshots deterministic:

- pass fixed UI state
- freeze clocks and animation progress when possible
- use local painters/bitmaps or fake image loaders
- avoid live network, current time, random IDs, and locale-dependent text unless controlled

## Fake Images And Platform Services

If image content is not the behavior under test, fake the loader and assert the requested model if useful. If the image appearance matters, pass a deterministic local painter or bitmap.

For platform services, prefer a fake interface passed through the state holder or preview/test wrapper. Do not make a plain UI test depend on real permissions, native controllers, system clocks, share sheets, or network state.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Full app graph to test an error row | Test the plain UI composable with `state = Error` |
| Mock ViewModel to verify a click | Pass a callback and assert it was invoked |
| Screenshot test for text presence | Use semantics assertion |
| Semantics test for padding, color, or focus ring | Use screenshot test |
| Test tags everywhere | Prefer text, content description, role, selected/focused semantics |
| Real image loading, network, clock, or random data | Fake or freeze the source |
| Keyboard/TV UI tested only with `performClick()` | Send key input and assert focus; see `focus.md` |

## Primary Sources

- `https://developer.android.com/develop/ui/compose/testing`
- `https://developer.android.com/develop/ui/compose/testing/apis`
- `https://developer.android.com/develop/ui/compose/tooling/previews`
- `https://developer.android.com/develop/ui/compose/accessibility`
