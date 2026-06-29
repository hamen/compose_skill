# Navigation

Use Navigation 3 for new code. For projects still on Navigation 2, use the type-safe destinations that landed in `2.8`. **Do not write new code against string routes.**

## Decision Table

Start here. Pick the row that matches the situation, then read the matching section below.

| You are doing… | Use |
|---|---|
| New multi-screen app, you own the back stack as state | Navigation 3 (`androidx.navigation3`) + `NavDisplay` |
| One screen with internal modes (wizard steps, tabs, expand/collapse) | plain state (`var step by rememberSaveable { … }`) — **no nav library** |
| Existing Nav2 codebase you cannot migrate yet | Nav2 type-safe destinations (`composable<T>`, `2.8+`) |
| New screen on a Nav2 project mid-migration | still Nav2 type-safe — convert one graph at a time, **never** string routes |
| Adaptive layout: list + detail side-by-side on large screens, stacked on phones | Nav3 `ListDetailSceneStrategy` |
| Default one-screen-at-a-time presentation | Nav3 `SinglePaneSceneStrategy` (the default) |
| Passing an argument to a destination | a field on the `@Serializable` `NavKey` data class — never interpolate a string |
| Navigating in response to a ViewModel result | expose `Flow<NavEvent>`, collect in a route `LaunchedEffect` — **never** inject `backStack` |
| Scoping a `viewModel()` to one entry | `rememberViewModelStoreNavEntryDecorator()` after the default saveable-state decorator |
| Intercepting back (unsaved-changes guard) | `BackHandler` on that screen only — otherwise rely on `NavDisplay`'s `onBack` |
| Entering from a deep link | resolve the intent to a destination key and push it; the destination must be self-sufficient |

**LLM tell:** reaching for `NavController` / `NavHost` / `navigate(route)` in new Nav3 code, or pulling in a navigation library for a single screen whose "navigation" is really internal state. Nav3 owns the back stack as plain state; a self-contained screen needs no library at all.

<https://developer.android.com/guide/navigation/navigation-3>

## Navigation 3 — The Model

Navigation 3 (`androidx.navigation3`) flips the model. Instead of a framework-owned nav graph with routes and arguments, you own the back stack as plain state:

```kotlin
@Serializable data object Home : NavKey
@Serializable data class Profile(val userId: String) : NavKey
@Serializable data class Settings(val section: String? = null) : NavKey

@Composable
fun App() {
    val backStack = rememberNavBackStack<NavKey>(Home)

    NavDisplay(
        backStack = backStack,
        onBack = { backStack.removeLastOrNull() },
        entryProvider = entryProvider {
            entry<Home> {
                HomeScreen(
                    onOpenProfile = { userId -> backStack.add(Profile(userId)) },
                    onOpenSettings = { backStack.add(Settings()) },
                )
            }
            entry<Profile> { key ->
                ProfileScreen(userId = key.userId, onBack = { backStack.removeLastOrNull() })
            }
            entry<Settings> { key ->
                SettingsScreen(section = key.section, onBack = { backStack.removeLastOrNull() })
            }
        },
    )
}
```

Core shifts from Nav2:

- **Back stack is plain state.** `backStack.add(key)` to push, `backStack.removeLastOrNull()` to pop. No `NavController.navigate(route)`.
- **Destinations are `@Serializable` data classes / objects** that implement `NavKey`. Arguments are class fields. No string interpolation.
- **Transitions, predictive back, and scenes are first-class.** The `Scene` strategies (`SinglePaneSceneStrategy`, `ListDetailSceneStrategy`, etc.) decide which entries to show together on a given window size.
- **Entries are decorated with owners and state support.** `NavDisplay` includes `rememberSaveableStateHolderNavEntryDecorator()` by default so `rememberSaveable` works inside entries. Per-entry `ViewModelStoreOwner` is added explicitly with `rememberViewModelStoreNavEntryDecorator()` from `androidx.lifecycle:lifecycle-viewmodel-navigation3`.

### Guardrails For Nav3

- Always declare destinations as top-level `@Serializable` types. Inline anonymous keys break `rememberSaveable` and process death restore.
- Pass domain values as **fields of the destination**, not captured by callbacks. Captured values lose process-death safety.
- Do not put `@Composable` functions or lambdas inside destination data. Destinations must be serializable.
- For navigating from a ViewModel, expose a `Channel<NavEvent>` or `Flow<NavEvent>` and collect it from a `LaunchedEffect` in the route — do **not** inject `backStack` into the ViewModel.
- If an entry calls `viewModel()` and you want it scoped to that `NavEntry`, add `rememberViewModelStoreNavEntryDecorator()` after the default saveable-state decorator.
- Use `entryProvider { ... }` to map destination type → composable. Each entry accepts the key so you can read its fields.

### Predictive Back

Navigation 3 supports predictive back out of the box when `NavDisplay` is used and `onBack` pops the stack. To customize the predictive animation, use `Scene` strategies or override the `TransitionSpec`. Do not wire up your own `BackHandler` inside destinations unless the screen needs to intercept back (e.g. unsaved changes dialog).

## Navigation 2 — Type-Safe Destinations (2.8+)

If the project is still on Navigation 2 and cannot move to Nav3 yet:

```kotlin
@Serializable data object Home
@Serializable data class Profile(val userId: String)

@Composable
fun App() {
    val navController = rememberNavController()
    NavHost(navController, startDestination = Home) {
        composable<Home> {
            HomeScreen(onOpenProfile = { id -> navController.navigate(Profile(id)) })
        }
        composable<Profile> { backStackEntry ->
            val profile = backStackEntry.toRoute<Profile>()
            ProfileScreen(userId = profile.userId, onBack = { navController.popBackStack() })
        }
    }
}
```

Guardrails:

- Never write `composable("profile/{id}")` for new screens. If the project has existing string routes, migration is incremental — convert one graph at a time.
- Do not pack complex arguments into strings. Use the `@Serializable` class; Nav2.8 serializes it into the back stack entry state.
- Prefer `navController.navigate(Profile(id)) { launchSingleTop = true }` when the same destination should not stack multiple instances.

## Common Mistakes

- **Navigating inside a composition.** Call sites should be callbacks wired to events, not computed during composition. `onClick = { navController.navigate(...) }` is right. Calling `navController.navigate(...)` in the composable body is wrong and will fire on every recomposition.
- **Capturing `navController` in a ViewModel.** Navigation is a UI concern; keep the controller in the UI layer. Expose a `Flow<NavEvent>` from the ViewModel.
- **Using `BackHandler` everywhere.** Only use it when the screen genuinely needs to intercept back (confirm unsaved changes, close a search field). Otherwise the framework's back handling is fine.
- **Hoisting nav state through multiple layers.** Pass `onXxx` callbacks — not the `NavController` — to child composables.
- **Using the old Accompanist Navigation Animation library.** Nav2.7+ has transition parameters on `composable(...)`; Nav3 has transitions at the `NavDisplay` / `Scene` level. `accompanist-navigation-animation` is retired.

## Deep Links

- Nav3: register deep links on the `entry` via the scene strategies, or handle them in a top-level `LaunchedEffect(intent)` that resolves the intent to a destination key and pushes it.
- Nav2: `deepLinks = listOf(navDeepLink { uriPattern = "..." })` on each `composable`.

In both cases, the destination must be **self-sufficient** when entered from a deep link — do not assume previous destinations are on the stack.

## Testing

- Each destination should be Compose-previewable on its own by passing a constructed state object. If your destination only works when `backStack` or `navController` is non-null, refactor — the screen composable should take callbacks, and only the route wrapper should know about navigation.
- In tests, pass a fake `backStack` (Nav3) or a `TestNavHostController` (Nav2).

## Primary Sources

- `https://developer.android.com/guide/navigation/navigation-3` — Navigation 3 overview
- `https://developer.android.com/guide/navigation/navigation-3/scenes` — scene strategies
- `https://developer.android.com/guide/navigation/use-graph/navigate#type-safe` — Nav2 type-safe destinations
- `https://developer.android.com/develop/ui/compose/navigation` — Compose + Nav2
