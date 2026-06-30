# Navigation

Use Navigation 3 for new code. For projects still on Navigation 2, use the type-safe destinations that landed in `2.8`. **Do not write new code against string routes.**

> Navigation 3 (`androidx.navigation3`) is still in **alpha** (`1.x-alpha`) — it is what Android recommends for new Compose navigation, but pin the version and expect some API churn. The shapes below are verified against the official [nav3-recipes](https://github.com/android/nav3-recipes) samples.

## Decision Table

Start here. Pick the row that matches the situation, then read the matching section below.

| You are doing… | Use |
|---|---|
| New multi-screen app, you own the back stack as state | Navigation 3 (`androidx.navigation3`) + `NavDisplay` |
| One screen with internal modes (wizard steps, in-screen tabs, expand/collapse) | plain state (`var step by rememberSaveable { … }`) — **no nav library**. (Bottom-nav with a separate back stack per tab is multi-destination → Nav3, not this row.) |
| Existing Nav2 codebase you cannot migrate yet | Nav2 type-safe destinations (`composable<T>`, `2.8+`) |
| New screen on a Nav2 project mid-migration | still Nav2 type-safe — convert one graph at a time, **never** string routes |
| **Nav3:** adaptive layout, list + detail side-by-side on large screens, stacked on phones | `ListDetailSceneStrategy` |
| **Nav3:** default one-screen-at-a-time presentation | `SinglePaneSceneStrategy` (the default) |
| Passing an argument to a destination | **Nav3:** a field on the `@Serializable` `NavKey` data class — never interpolate a string. **Nav2 type-safe:** fields on the `@Serializable` destination, read via `toRoute()` |
| Navigating off a ViewModel signal | the route observes the VM and calls `backStack.add` / `removeLastOrNull` **itself** — **don't** inject the `backStack` (Nav3) / `navController` (Nav2) into a screen/feature VM (a dedicated app-level nav holder that owns the stack is a separate pattern) |
| Passing a result back from another screen (Nav3) | the `ResultEventBus` (`rememberResultEventBusNavEntryDecorator()`); consume as state (`conflateAsState`) or as a one-shot (`ResultEffect { … }`) |
| **Nav3:** scoping a `viewModel()` to one entry | `rememberViewModelStoreNavEntryDecorator()` after the default saveable-state decorator (Nav2 already scopes `viewModel()` to the `NavBackStackEntry` — do not add this there) |
| Intercepting back (unsaved-changes guard) | `BackHandler` on that screen only — otherwise rely on the default nav-host back handling (`NavDisplay.onBack` for Nav3 / `NavHost` for Nav2) |
| Entering from a deep link | **Nav3:** resolve the intent to a destination key, then seed it as the start key (or push it for a new intent); the destination must be self-sufficient. **Nav2:** `navDeepLink { … }` (see "Deep Links" below) |

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
    val backStack = rememberNavBackStack(Home)  // vararg of NavKey; no type argument

    NavDisplay(
        backStack = backStack,
        onBack = { backStack.removeLastOrNull() },
        entryProvider = entryProvider {
            entry<Home> {
                HomeScreen(
                    // the child screen wraps its actual Button onClick in
                    // dropUnlessResumed { … } (see guardrails) before calling these
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
- **Transitions, predictive back, and scenes are first-class.** Scene strategies decide which entries to show together on a given window size. Pass them via `NavDisplay(sceneStrategies = listOf(rememberListDetailSceneStrategy<NavKey>()))` and tag entries with `metadata = ListDetailSceneStrategy.listPane()` / `.detailPane()` (and `.extraPane()` for a third pane). The adaptive list-detail strategy lives in `androidx.compose.material3.adaptive:adaptive-navigation3`; `SinglePaneSceneStrategy` is the default.
- **Entries are decorated with owners and state support.** `NavDisplay` includes `rememberSaveableStateHolderNavEntryDecorator()` by default so `rememberSaveable` works inside entries. Per-entry `ViewModelStoreOwner` is added explicitly with `rememberViewModelStoreNavEntryDecorator()` from `androidx.lifecycle:lifecycle-viewmodel-navigation3`.

### Guardrails For Nav3

- Always declare destinations as top-level `@Serializable` types. Inline anonymous keys break `rememberSaveable` and process death restore.
- Pass domain values as **fields of the destination**, not captured by callbacks. Captured values lose process-death safety.
- Do not put `@Composable` functions or lambdas inside destination data. Destinations must be serializable.
- `rememberNavBackStack(...)` takes a **`vararg` of `NavKey`** and returns `NavBackStack<NavKey>` — it is **not** generic. Call it `rememberNavBackStack(Home)`, never `rememberNavBackStack<NavKey>(Home)` (that does not compile). If you genuinely need `NavBackStack<MyKey>`, write your own thin wrapper; the library overload is `NavKey`-typed.
- **Don't make a feature/screen ViewModel own or mutate the back stack.** Navigation triggered by a ViewModel should flow the other way: the route observes the VM's state and calls `backStack.add` / `removeLastOrNull` itself — do not inject `backStack` into a screen VM as a navigate-command surface. (A dedicated app-level navigation holder that *owns* the stack — like the recipes' `Navigator` / `NavigationState` — is a separate, legitimate pattern; the save-state doc also covers persisting a back stack from such a holder.)
- **Supplying `entryDecorators` replaces *all* built-in decorators.** Whenever you pass a custom list — for results, per-entry ViewModels, anything — re-add the defaults you still rely on, starting with `rememberSaveableStateHolderNavEntryDecorator()`. Without it, `rememberSaveable` inside entries silently stops working. (The official result recipes list only the result decorator precisely because those screens don't use `rememberSaveable` — don't copy that omission into a screen that does.)
- To hand a **result** back from a child screen, Nav3 provides a `ResultEventBus`: add `rememberResultEventBusNavEntryDecorator()` to `entryDecorators` (alongside `rememberSaveableStateHolderNavEntryDecorator()` per the rule above), call `LocalResultEventBus.current.sendResult(value)` on the producer, and consume it on the target either as state (`LocalResultEventBus.current.conflateAsState(...)`) or as a one-shot (`ResultEffect<T> { … }`). Both forms are official. Note the `conflateAsState` value is the latest result **within the current navigation lifetime — it is not retained across process death**; for an outcome that must survive that, persist it in a state holder (`flows.md`).
- Wrap navigation triggered by a tap in `dropUnlessResumed { backStack.add(...) }` (from `androidx.lifecycle.compose`) so a queued click cannot navigate from a screen that has already left `RESUMED` — every official recipe does this.
- To scope a `viewModel()` to one `NavEntry`, add `rememberViewModelStoreNavEntryDecorator()` (from `androidx.lifecycle.viewmodel.navigation3`) to `entryDecorators` — again alongside `rememberSaveableStateHolderNavEntryDecorator()` per the replace-the-defaults rule.
- Use `entryProvider { entry<Key> { … } }` to map destination type → composable. Each entry accepts the key so you can read its fields.

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
- **Capturing the `navController` (Nav2) or `backStack` (Nav3) in a screen/feature ViewModel.** Navigation is a UI concern; the back stack is mutated in the route, not a screen VM. Have the UI observe the VM (its state, or Nav3's `ResultEventBus` for a returned result) and navigate itself. (A dedicated app-level navigation holder that *owns* the stack is fine — see the guardrail above.)
- **Using `BackHandler` everywhere.** Only use it when the screen genuinely needs to intercept back (confirm unsaved changes, close a search field). Otherwise the framework's back handling is fine.
- **Hoisting nav state through multiple layers.** Pass `onXxx` callbacks — not the `NavController` (Nav2) or `backStack` (Nav3) — to child composables.
- **Using the old Accompanist Navigation Animation library.** Nav2.7+ has transition parameters on `composable(...)`; Nav3 has transitions at the `NavDisplay` / `Scene` level. `accompanist-navigation-animation` is retired.

## Deep Links

- Nav3: resolve the intent's `Uri` to a destination `NavKey` (the recipes decode it with `kotlinx.serialization`) and seed it as the start key — `rememberNavBackStack(resolvedKey)` — or, for a new intent while running, push it from `onNewIntent` / a top-level effect. There is no deep-link DSL; you own the intent → key mapping. (Scene strategies decide *which* entries render together for a given window size — they do not register URIs.)
- Nav2: `deepLinks = listOf(navDeepLink { uriPattern = "..." })` on each `composable`.

In both cases, the destination must be **self-sufficient** when entered from a deep link — do not assume previous destinations are on the stack.

## Testing

- Each destination should be Compose-previewable on its own by passing a constructed state object. If your destination only works when `backStack` or `navController` is non-null, refactor — the screen composable should take callbacks, and only the route wrapper should know about navigation.
- In tests, pass a fake `backStack` (Nav3) or a `TestNavHostController` (Nav2).

## Primary Sources

- `https://developer.android.com/guide/navigation/navigation-3/basics` — Nav3 basics (`NavDisplay`, `entryProvider`, back stack as state)
- `https://developer.android.com/guide/navigation/navigation-3/save-state` — `rememberNavBackStack`, `NavKey` + `@Serializable`, the `rememberSaveableStateHolderNavEntryDecorator` / `rememberViewModelStoreNavEntryDecorator` decorators
- `https://developer.android.com/guide/navigation/navigation-3/scenes` — scene strategies (`SinglePaneSceneStrategy`, `ListDetailSceneStrategy`); they group entries by window size, they do not register deep links
- `https://developer.android.com/guide/navigation/use-graph/navigate#type-safe` — Nav2 type-safe destinations
- `https://developer.android.com/develop/ui/compose/navigation` — Compose + Nav2
- `https://github.com/android/nav3-recipes` — official Nav3 sample recipes (basic, saveable, DSL, per-entry ViewModel, list-detail scenes, results via `ResultEventBus`, deep links). The shapes in this reference are checked against these.
