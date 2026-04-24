# Composable API Guidelines

These rules come from the AndroidX composable component API guidelines. They are what turns "a composable that works" into "a composable other developers can drop in without surprises".

This file covers what to enforce when writing a new reusable composable or reviewing one.

## Naming

- Composables that **render UI** are named with a noun phrase in `PascalCase`: `UserCard`, `PriceTag`, `CheckoutButton`. They read like the thing they produce.
- Composables that are **effects** (return `Unit`, perform a side effect, do not emit UI) are named with a verb phrase in `camelCase` starting with the action: `launchScreenViewEvent`, `observeOrientation`. Rare outside of libraries — most `@Composable` functions are UI.
- Factories that return a remembered object (not a composable rendering) are named `rememberXxx`: `rememberAppBarState`, `rememberPagerState`. Always use `remember` inside, return the state object.
- Scope/receiver types expose nested composables in `camelCase`: `LazyListScope.items(...)` (in this file, naming follows the standard extension-function rules).

## Parameter Order

```kotlin
@Composable
fun UserCard(
    // 1. Required data — no defaults
    user: User,
    onClick: () -> Unit,

    // 2. The modifier — always second after required data (or first optional), typed and defaulted
    modifier: Modifier = Modifier,

    // 3. Other optional parameters with defaults
    showBadge: Boolean = false,
    onLongClick: (() -> Unit)? = null,

    // 4. Slot parameters — trailing @Composable lambdas last
    trailingAction: (@Composable () -> Unit)? = null,
    content: @Composable ColumnScope.() -> Unit,
) { ... }
```

Rules:

- **Modifier is the first optional parameter.** Callers that do `UserCard(user, onClick) { ... }` should not need named arguments to drop in a modifier.
- **Exactly one `content` slot** named `content` — if the composable has a primary content area. For composables with multiple slots (`TopAppBar(title, actions, navigationIcon)`), each gets its own name.
- **Slot lambdas last.** Callers use trailing-lambda syntax for the `content`.
- **Do not default `content`** to `null` — a `null` slot is a smell. If the slot is optional, make it `(@Composable () -> Unit)? = null` and branch; usually it is clearer to provide a default that renders nothing or a meaningful default.

## Slot APIs

Slots are how composables become composable. Prefer slots over enums / booleans for content variation.

```kotlin
// Weak — caller has no control over what the leading element is
@Composable
fun Row(
    primaryText: String,
    secondaryText: String?,
    icon: ImageVector?,
    isSelected: Boolean,
) { ... }

// Strong — caller supplies whatever they want
@Composable
fun Row(
    headline: @Composable () -> Unit,
    modifier: Modifier = Modifier,
    supportingContent: (@Composable () -> Unit)? = null,
    leadingContent: (@Composable () -> Unit)? = null,
    trailingContent: (@Composable () -> Unit)? = null,
) { ... }
```

Rules:

- For a library-grade component, **two to five slots is normal**. A single `content` slot is fine for containers.
- Name the slots after their role, not their appearance: `leading`, `trailing`, `supporting`, `headline`, `action` — not `left`, `right`, `top`.
- Give slots a `Scope` receiver when the slot's composables need access to state from the parent (e.g. `RowScope`, `BoxScope`, or a custom `MyComponentScope`).

## Defaults

Every optional parameter has a default. Defaults should be:

- **Sensible for the most common case.** `showBadge: Boolean = false` is fine. `showBadge: Boolean = true` is questionable.
- **Exposed through a `Defaults` object** when the defaults are configurable at the theme level. M3 uses this: `ButtonDefaults.buttonColors()`, `TextFieldDefaults.shape`. If the default depends on theming, expose it through a `Defaults` companion.
- **Not expensive to evaluate.** A default that constructs a state object on every invocation allocates on every call — either hoist to a constant or use `remember`.

## State Hoisting Shape

Library composables follow the stateless + stateful pair pattern:

```kotlin
// Stateless — caller owns state
@Composable
fun Checkbox(
    checked: Boolean,
    onCheckedChange: ((Boolean) -> Unit)?,
    modifier: Modifier = Modifier,
    enabled: Boolean = true,
    // ...
) { ... }

// Stateful convenience — for the common case where caller does not need the state elsewhere
@Composable
fun Checkbox(
    modifier: Modifier = Modifier,
    initiallyChecked: Boolean = false,
    enabled: Boolean = true,
    // ...
) {
    var checked by rememberSaveable { mutableStateOf(initiallyChecked) }
    Checkbox(
        checked = checked,
        onCheckedChange = { checked = it },
        modifier = modifier,
        enabled = enabled,
    )
}
```

Rules:

- **The stateless overload is the primary API.** The stateful convenience is optional and only makes sense when the component is genuinely useful without the caller observing state.
- **Never expose `MutableState<T>` in the API.** If you must expose mutable state, expose a state-holder class with `State<T>` properties and mutator functions. See `references/state.md`.
- **Event callbacks are named `onXxx`**: `onClick`, `onCheckedChange`, `onSelectionChange`. They receive the new value.
- **Do not use Boolean `isXxx` names for observable state.** `selected: Boolean` paired with `onSelectionChange: (Boolean) -> Unit` is the convention.

## Scope Receivers For Nested DSLs

If a composable exposes child composables that only make sense in its context, publish a scope receiver:

```kotlin
@LayoutScopeMarker
interface PagerScope {
    val currentPage: Int
    fun Modifier.pagerItem(page: Int): Modifier
}

@Composable
fun Pager(
    state: PagerState,
    modifier: Modifier = Modifier,
    content: @Composable PagerScope.() -> Unit,
) { ... }
```

Rules:

- Always apply `@LayoutScopeMarker` (or the equivalent `DslMarker`) to prevent implicit receiver shadowing bugs.
- Do **not** expose members on the scope that do not require the scope. `currentPage` on `PagerScope` makes sense; a helper that does not reference the pager should be a free function.

## Compositional Purity

- A composable should be **deterministic** given its inputs plus captured `CompositionLocal`s. Calling it twice with the same inputs must produce the same UI tree.
- **Do not** perform I/O, read the clock, or generate random values during composition. Wrap them in an effect and expose the result as state.
- **Do not** mutate parameters. Parameters are read-only inputs.

## `CompositionLocal` Usage

Only use `CompositionLocal` for values that are:

1. **Tree-scoped** — meaningful for an entire subtree, not just one component.
2. **Have a sensible default** — default value makes preview and testing trivial.
3. **Not flow-critical** — changing the value does not break the UI's meaning, only its appearance.

Good: `LocalContentColor`, `LocalTextStyle`, `LocalContext`, `LocalDensity`.

Bad:

- `LocalOnClick` — use an explicit `onClick` parameter.
- `LocalViewModel` — use explicit parameters or a DI container.
- `LocalSnackbar` — use a `SnackbarHostState` passed explicitly or through a route-level scope.

Rule of thumb: if a CompositionLocal's value is expected to differ between components, it is the wrong shape.

## Previews

Every reusable composable should be previewable without a live data source. If it requires a `ViewModel` or a coroutine scope to render, the API is wrong.

```kotlin
@Preview
@Composable
private fun UserCardPreview() {
    AppTheme {
        UserCard(
            user = User.sample(),
            onClick = {},
        )
    }
}
```

Rules:

- Previews are `private`. Do not pollute the public API.
- Use `@PreviewLightDark`, `@PreviewFontScale`, `@PreviewScreenSizes` multipreview annotations to cover variants cheaply.
- Construct sample data via factory methods or a `Previews.kt` file in the same module. Do not rely on network data.

## Accessibility

Not scored here, but the component API guidelines make these hard requirements:

- `Icon` takes a `contentDescription`. `null` is a valid value **only** if the icon is purely decorative and an adjacent `Text` describes the element.
- Interactive composables (`clickable`, `Button`, `IconButton`, `Switch`) have a `Role` set appropriately when they are not the default role.
- Touch targets are at least `48dp` on both axes. Use `Modifier.minimumInteractiveComponentSize()` or wrap in `IconButton`.

## Dark Mode Correctness — Hard-Coded Colors On Theme Backgrounds

A cross-cutting rule the skill enforces even though full theming scoring defers to the `material-3` skill: **never pair a hard-coded text / icon color with a theme-derived background**. The two resolve independently, so "looks fine in light mode" is not a guarantee for dark mode.

```kotlin
// Wrong — background shifts in dark mode; text stays black and becomes unreadable
val blended = Color(
    ColorUtils.blendARGB(
        MaterialTheme.colorScheme.primary.toArgb(),
        MaterialTheme.colorScheme.surface.toArgb(),
        0.6f,
    )
)
Text(title, color = Color.Black, modifier = Modifier.background(blended))

// Right — text color reads from the same theming source as the background
Text(title, color = MaterialTheme.colorScheme.onSurface, modifier = Modifier.background(blended))
```

Rules:

- Text / icon color over a `MaterialTheme.colorScheme.*` (or blended) background must come from the matching `on*` role (`onSurface`, `onPrimary`, `onPrimaryContainer`, etc.). Those roles are contrast-paired by Material.
- `Color.Black` / `Color.White` literals as foreground, paired with a theme-driven sibling background, are dark-mode regressions waiting to ship.
- Raw ARGB literals (`Color(0xFF...)`) as foreground over theme backgrounds have the same problem. Brand colors that are intentionally theme-independent (logo fill, a fixed-identity pill) are fine — but the text over them should still pick up the brand color's paired on-color, not `Color.Black`.

### Grep triggers for this rule

- `Color\.Black\b`, `Color\.White\b` — inspect every hit; unsafe if a sibling element reads `MaterialTheme.colorScheme`
- `color\s*=\s*Color\(0x` as a foreground on an element whose background reads `colorScheme.*`
- `blendARGB\(` — a blended theme background almost always pairs with theme text, not a literal

## Documentation

Every public composable should have a KDoc that includes:

- One-line summary of what it renders.
- A sample usage block with `@sample`.
- Parameter documentation, especially for slots (describe what each slot is for and what scope it receives).

```kotlin
/**
 * Displays a compact user card with avatar, name, and an optional trailing action.
 *
 * @param user the user to render.
 * @param onClick invoked when the card is tapped.
 * @param modifier the [Modifier] applied to the outermost layout.
 * @param trailingAction optional slot displayed on the right edge of the card.
 *
 * @sample com.example.samples.UserCardSample
 */
@Composable
fun UserCard(...) { ... }
```

## Grep Triggers

- Composables with a slot named `content` but no `content: @Composable () -> Unit` at the **end** of the parameter list — parameter order regression.
- `var modifier` anywhere — should be `val` (or not a parameter at all).
- `modifier: Modifier` without `= Modifier` — missing default.
- `MutableState<T>` in a public composable parameter type — expose `State<T>` or a callback pair.
- `fun rememberXxx` without calling `remember` internally.

## Primary Sources

- `https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md`
- `https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-api-guidelines.md`
- `https://developer.android.com/develop/ui/compose/api-guidelines`
- `https://developer.android.com/develop/ui/compose/compositionlocal`
