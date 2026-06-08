# Deprecated and Soft-Deprecated Compose APIs

The LLM's training data is full of Compose code from 2021–2023. Most of these APIs still compile, some only emit a deprecation warning, but none of them are what you should write today. This file is the hit list.

When you see any of these in reviewed code — or are about to write any of them — switch to the replacement.

## Accompanist — All Effectively Retired

The Accompanist project was absorbed into AndroidX between 2023 and 2024. The libraries below are **deprecated or read-only** and should not be added to new code.

| Accompanist library | Replacement |
|---|---|
| `accompanist-pager` | `HorizontalPager` / `VerticalPager` in `androidx.compose.foundation.pager` |
| `accompanist-pager-indicators` | Build a simple `Row` of dots driven by `PagerState.currentPage`, or use Material 3 `PageIndicator` on the platforms that ship it |
| `accompanist-swiperefresh` | `PullToRefreshBox` in `androidx.compose.material3.pulltorefresh` |
| `accompanist-flowlayout` | `FlowRow` / `FlowColumn` in `androidx.compose.foundation.layout` |
| `accompanist-systemuicontroller` | `ComponentActivity.enableEdgeToEdge()` plus `WindowInsets.*` for per-surface tweaks |
| `accompanist-insets` | `WindowInsets.*` + `Modifier.windowInsetsPadding(...)` (in `androidx.compose.foundation.layout`) |
| `accompanist-placeholder` | Community alternatives or a custom `Modifier.drawBehind` / `Modifier.shimmer` |
| `accompanist-navigation-animation` | `NavHost` from `androidx.navigation:navigation-compose` (2.7+) has built-in enter/exit transition params; for Nav3, transitions are first-class |

If a project still depends on these, the migration is usually mechanical. Flag it and prioritize `systemuicontroller` and `swiperefresh` first — the replacements have better behavior around insets and IME.

## Collect Flow With Lifecycle

**Soft-deprecated in UI code:** `Flow<T>.collectAsState()`

```kotlin
// Wrong — keeps collecting while the screen is in the back stack
val user by viewModel.user.collectAsState()

// Right
val user by viewModel.user.collectAsStateWithLifecycle()
```

`collectAsStateWithLifecycle()` is in `androidx.lifecycle:lifecycle-runtime-compose`. It respects `Lifecycle.State.STARTED` by default, which is what you almost always want for a screen. Use `collectAsState()` only for state that must update while the UI is not visible (rare — usually that state belongs to the ViewModel, not the composition).

For StateFlow specifically, `collectAsStateWithLifecycle()` has a no-initial-value overload that reuses the flow's current value, so there's no dummy initial to think about.

## Animation Phase APIs

**Soft-deprecated pattern:** reading an animated `.value` and piping it into a state-reading modifier.

```kotlin
// Wrong — recomposes on every animation frame
val offsetX by animateDpAsState(targetX, label = "x")
Modifier.offset(x = offsetX)

// Right — defers the read to layout
val offsetX by animateDpAsState(targetX, label = "x")
Modifier.offset { IntOffset(offsetX.roundToPx(), 0) }
```

Same logic for `alpha`, `rotate`, `scale`, `padding`, and any other modifier that has a lambda-form variant. `Modifier.graphicsLayer { ... }` is the generic escape hatch for alpha / scale / rotation / translation driven by state.

**Deprecated:** `LazyItemScope.animateItemPlacement(...)`  
**Replacement:** `LazyItemScope.animateItem(...)` (Compose foundation `1.7+`). The new API also animates visibility changes, not just placement.

**Deprecated:** creating `Animatable(...)` directly in a composable body without `remember { ... }`.  
**Replacement:** hoist into a remembered field or use `animate*AsState` / `rememberInfiniteTransition`.

## Material — M2 → M3

Where a project has adopted Material 3, the Material 2 (`androidx.compose.material:material`) equivalents should not be mixed in. Most teams ship a slow migration — flag M2 usage one component at a time.

| Material 2 | Material 3 |
|---|---|
| `androidx.compose.material.Scaffold` | `androidx.compose.material3.Scaffold` |
| `BottomNavigation` / `BottomNavigationItem` | `NavigationBar` / `NavigationBarItem` |
| `TopAppBar` (M2) | `TopAppBar` / `CenterAlignedTopAppBar` / `MediumTopAppBar` / `LargeTopAppBar` (M3) with `TopAppBarScrollBehavior` |
| `BackdropScaffold` | No direct M3 equivalent — rebuild with `ModalBottomSheet` or a custom layout |
| `Divider` | `HorizontalDivider` / `VerticalDivider` |
| `ModalBottomSheetLayout` | `ModalBottomSheet` |
| `androidx.compose.material.icons.*` | Same icon package works, but prefer the extended icon set `androidx.compose.material.icons.Icons.Outlined.*` / `Rounded.*` where your design system picks a specific style |

### Material 3 — Within M3, Watch For

**Deprecated progress signature:** `LinearProgressIndicator(progress = 0.5f)` / `CircularProgressIndicator(progress = 0.5f)` as a raw `Float`.  
**Replacement:** the lambda overload `LinearProgressIndicator(progress = { 0.5f })`. Lambda-form lets the progress read be deferred, and matches the other M3 APIs that took the same shape change.

**Deprecated slot order** on older `DropdownMenuItem`: positional `(text: String, onClick: () -> Unit, ...)`.  
**Replacement:** slot-based `DropdownMenuItem(text = { Text(...) }, onClick = ..., leadingIcon = { ... })`.

## Navigation

**Soft-deprecated pattern:** string-route navigation graphs on Navigation 2.

```kotlin
// Legacy
NavHost(navController, startDestination = "profile/123") {
    composable("profile/{id}") { backStackEntry ->
        val id = backStackEntry.arguments?.getString("id")
        ProfileScreen(id = id)
    }
}
```

**Replacement paths:**

- **Navigation 2.8+ with type-safe destinations:** declare `@Serializable` data classes / objects as routes.
- **Navigation 3** (`androidx.navigation3`): scene-based model where you own the back stack as plain state. See `references/navigation.md`.

## Lifecycle Effects — Use The Purpose-Built APIs

**Soft-deprecated pattern:** `DisposableEffect` + `LifecycleEventObserver` to run something on `ON_START` / `ON_RESUME`.

```kotlin
// Old pattern
val owner = LocalLifecycleOwner.current
DisposableEffect(owner) {
    val observer = LifecycleEventObserver { _, event ->
        if (event == Lifecycle.Event.ON_START) viewModel.refresh()
    }
    owner.lifecycle.addObserver(observer)
    onDispose { owner.lifecycle.removeObserver(observer) }
}

// Replacement — lifecycle-runtime-compose 2.8+
LifecycleStartEffect(Unit) {
    viewModel.refresh()
    onStopOrDispose { }
}
```

`LifecycleResumeEffect` for RESUME/PAUSE. Requires `androidx.lifecycle:lifecycle-runtime-compose:2.8.0+`. Full details in `references/effects.md`.

## Lifecycle Import Drift

`LocalLifecycleOwner` used to live in `androidx.compose.ui.platform`. The modern import is `androidx.lifecycle.compose.LocalLifecycleOwner`. The `compose.ui.platform` version is deprecated and delegates through. When touching a file, prefer the new import; do not chase existing files repo-wide unless asked.

## Android 12+ Splash Icon Blur

### Why it happens

The Android 12+ splash screen (the platform `SplashScreen` API, and the `androidx.core:core-splashscreen` compat library) shows a single centered icon driven by the theme attribute `windowSplashScreenAnimatedIcon`. The official docs already say this icon should be an `AnimatedVectorDrawable`, and the platform's internals are why that matters.

Per the AOSP source quoted in the platform issue (`https://issuetracker.google.com/issues/520672537`), there are two render paths:

- An `AnimatedVectorDrawable` is `Animatable`, so the system draws it on a **SurfaceView at full size**. Crisp.
- Anything else — a static `<vector>`, `<adaptive-icon>`, bitmap, or `<layer-list>` — goes through an internal `ImmobileIconDrawable` optimization. That class pre-renders the icon into a bitmap at `starting_surface_default_icon_size` = **108 dp**, then scales that bitmap up to the visible icon size (**160 dp**, sometimes **192 dp**). Detail is thrown away at 108 dp *before* the upscale — that is why the otherwise crisp icon looks blurry.

Scope, from the issue: present **since Android 12 (API 31)**, only on devices at **XHDPI (320 dpi) or higher** (the `loadInDetail == false` branch; lower-density devices happen to render at the final size and look fine). As of this writing the bug is **still open and unfixed** — Google has it as Assigned / P2 / Needs Info and has not yet reproduced it. Re-check the issue before asserting a specific fixed-in API level.

### Sizing the art (so the wrapper isn't wasted)

The wrapper fixes the *raster path*; it does not fix art that was authored too small. For an animated splash icon the official guidance is a **432 dp** canvas (four times the 108 dp adaptive area) with the visible content living in the inner two-thirds (**288 dp**). Keep the `android:width`/`android:height`/`viewport*` of the wrapped vector consistent with that so the system isn't upscaling a tiny viewport.

### The fix — wrap the vector in an animated-vector (no animators needed)

The reporter states it directly: *"It's enough to use an `<animated-vector>` with no animators."* You do **not** need a real animation, a `<target>`, an `objectAnimator`, or a named group — being an `AnimatedVectorDrawable` is the only thing that flips the render path.

Recommended shape — works on every API ≥ 21, one copy of the artwork, no `-v31` split needed:

```xml
<!-- res/drawable/ic_splash_logo.xml — the REAL artwork; the only copy of the paths. -->
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="288dp" android:height="288dp"
    android:viewportWidth="288" android:viewportHeight="288">
    <path android:fillColor="#FF6750A4" android:pathData="M..." />
</vector>
```

```xml
<!-- res/drawable/ic_splash.xml — what the theme points at. No animators. -->
<animated-vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/ic_splash_logo" />
```

```xml
<!-- res/values/themes.xml -->
<item name="android:windowSplashScreenAnimatedIcon">@drawable/ic_splash</item>
```

Notes that make this actually work:

- **Avoid the self-reference trap.** The `<animated-vector>` must point `android:drawable` at a *different* resource name than the theme uses. Resource references resolve by qualifier at runtime, so an `<animated-vector android:drawable="@drawable/ic_splash">` saved as `res/drawable-v31/ic_splash.xml` would re-resolve `@drawable/ic_splash` to the v31 file itself — infinite recursion. Keeping the artwork under its own name (`ic_splash_logo`) sidesteps it entirely.
- **`-v31` split is optional.** If you specifically want a lighter static `<vector>` on API < 31, put the `<animated-vector>` only in `res/drawable-v31/` — it must still reference a separately-named artwork vector (never `@drawable/ic_splash`), for the recursion reason above. The audit looks for an `<animated-vector>` resolving on API 31+; either layout satisfies it.
- **If you *want* a real animation**, fine — add `<target>`s (each must name a real `<group>`/`<path>` in the wrapped vector or it logs `Cannot find target` and is dropped) and keep the start delay ≤ 166 ms and total ≤ ~1000 ms via `windowSplashScreenAnimationDuration`. But that's a design choice, not part of the blur fix.

### What not to do

Do not "fix" this with a Compose-only splash composable. The starting window is drawn by the system *before* any Compose content exists, so a Compose splash can't replace it — it can only add a second, later screen. Fix the Android resource.

## View Interop You Probably Do Not Need

- `AndroidView` is fine for genuine platform widgets (MapView, WebView, exoplayer PlayerView). Do not wrap a `TextView` just to render text.
- `AndroidViewBinding` is for reusing existing XML layouts incrementally — flag if a brand-new screen uses it.
- `ComposeView` inside XML is fine during migration; do not add new XML layouts to host `ComposeView` if the parent can just be a pure composable.

## Coroutines And Threading

**Deprecated patterns** (not by deprecation annotation, but by the Compose + Coroutines model):

- `Handler(Looper.getMainLooper()).postDelayed { ... }` inside composables → use `LaunchedEffect { delay(...); ... }`.
- `AsyncTask` (removed from SDK 30+), `Thread { ... }.start()` inside a composable, `Executors.newSingleThreadExecutor()` for UI side effects → always use a coroutine launched from the ViewModel or from a scope tied to composition.
- `runBlocking` in UI code — essentially always wrong.

## Composition-Local Lambdas

Putting a lambda in a `CompositionLocal` (e.g. `LocalOnClick` or `LocalShowSnackbar`) is not formally deprecated, but is explicitly called out in the Compose API guidelines as the wrong tool. Use an explicit parameter, a scope class, or a plain callback.

## Quick Grep Triggers

If you're scanning a repo, these patterns surface the largest share of API smells:

- `collectAsState\(` — should almost always be `collectAsStateWithLifecycle`
- `animateItemPlacement\(`
- `androidx\.compose\.material\.` — Material 2 imports
- `com\.google\.accompanist\.` — Accompanist imports
- `Divider\(` (without the `Horizontal`/`Vertical` prefix)
- `progress\s*=\s*\w+\s*\)` in `LinearProgressIndicator` / `CircularProgressIndicator`
- `Modifier\.composed\s*\{` — should be `Modifier.Node`
- `Handler\s*\(` in Compose files
- `\.value\b` from an animated `State<T>` feeding a non-lambda modifier argument
- `LifecycleEventObserver` inside a `DisposableEffect` — use `LifecycleStartEffect` / `LifecycleResumeEffect`
- `windowSplashScreenAnimatedIcon` in `res/values*/themes.xml` or `styles.xml` — if the referenced API 31+ drawable resolves to a static `<vector>` instead of `<animated-vector>`, wrap it in a no-op animated-vector to avoid Android 12+ splash icon blur

## Primary Sources

- `https://developer.android.com/develop/ui/compose` (canonical Compose)
- `https://developer.android.com/jetpack/androidx/releases/compose-material3` (M3 changelog)
- `https://google.github.io/accompanist/` (marks each library's migration target)
- `https://developer.android.com/develop/ui/compose/performance/bestpractices` (deferred reads)
- `https://developer.android.com/develop/ui/views/launch/splash-screen`
- `https://developer.android.com/reference/androidx/core/splashscreen/SplashScreen`
