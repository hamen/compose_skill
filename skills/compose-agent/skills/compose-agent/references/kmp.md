# Kotlin Multiplatform And Compose Multiplatform

Keep common APIs semantic and stable. Put platform mechanics behind small `expect` / `actual` declarations or fakeable interfaces, and keep Android, iOS, Desktop, and Wasm details out of `commonMain`.

Use this reference when common code or shared Compose UI needs:

- permissions, settings, intents, share sheets, deep links, haptics, biometrics, clipboard
- files, paths, clocks, locale, network reachability, sensors, crypto, media, maps, camera, native SDKs
- native platform views or Compose Multiplatform interop
- a decision between `expect` / `actual`, dependency injection, interfaces, or platform-specific screens

## Choose The Boundary

| Situation | Prefer |
|---|---|
| Simple compile-time platform specialization | `expect` / `actual` function, value, typealias, or leaf composable |
| Runtime dependencies, lifecycle ownership, fakes, multiple implementations | Common interface plus platform binding |
| Shared UI with one platform-specific leaf | Common composable calling an `expect` leaf or injected capability |
| Entire screen differs by platform | Separate platform screens behind a common navigation contract |
| Constants or resources differ | Semantic common API with actual values per platform |

## Keep Common APIs Semantic

Common code should describe what the product needs, not how one platform does it.

```kotlin
// Good: semantic common API.
expect fun currentRegion(): Region
```

```kotlin
// Bad: Android detail leaks into commonMain.
expect fun currentRegionFromAndroidLocale(context: Context): Region
```

The Android actual can use `Locale` APIs. The iOS actual can use Foundation APIs. Callers should not know.

## Keep Actuals Thin

Actual implementations translate a semantic API into platform calls. If the implementation needs an Activity, view controller, lifecycle owner, DI, or test fake, prefer an interface supplied by platform code:

```kotlin
// commonMain
interface ShareSheet {
    suspend fun shareText(text: String)
}
```

```kotlin
// androidMain
class AndroidShareSheet(
    private val activity: Activity,
) : ShareSheet {
    override suspend fun shareText(text: String) {
        val intent = Intent(Intent.ACTION_SEND)
            .setType("text/plain")
            .putExtra(Intent.EXTRA_TEXT, text)
        activity.startActivity(Intent.createChooser(intent, null))
    }
}
```

Define what `suspend` means. For many platform UI actions it means "the sheet was launched", not "the user completed the action".

If an actual accumulates product rules, move those rules back to common code and leave only platform translation in the actual.

## Compose-Specific Rules

- Keep platform-specific composables at leaf nodes.
- Any expected composable that emits UI still takes and forwards `modifier: Modifier = Modifier`.
- Avoid platform types in `commonMain` signatures: `Context`, `Activity`, Android resource IDs, `Uri`, `Bundle`, `UIViewController`, `NSBundle`, permission enums, and platform view types.
- Hide native view lifecycle inside the platform actual and use the right interop container (`AndroidView`, `UIKitView`, etc.).
- Do not launch platform work from a composable body. Use `LaunchedEffect`, `DisposableEffect`, `remember`, and stable keys exactly as in Android Compose.
- Keep previews and tests on plain common UI composables with fake platform services when possible.

## Interfaces vs `expect` / `actual`

Use `expect` / `actual` for simple compile-time APIs. Use interfaces when common code needs fakes, dependency injection, lifecycle ownership, or runtime choice.

```kotlin
// commonMain
interface Clipboard {
    suspend fun setText(text: String)
}
```

Platform modules bind `Clipboard` to Android, iOS, Desktop, or test implementations. Common tests use a fake.

## Common Mistakes

| Mistake | Fix |
|---|---|
| `commonMain` exposes Android/iOS types | Replace with semantic common types |
| `expect` function has one-platform-only parameters | Move details into the actual |
| Business branching duplicated in each actual | Move business rules to common code |
| One huge `Platform` expect object | Split by capability: `Clipboard`, `ShareSheet`, `Haptics` |
| Platform UI leaks high in the tree | Push platform-specific composable to a leaf |
| No fakeable boundary for common tests | Use an interface instead of direct `expect` call |

## Review Red Flags

- common code imports platform packages
- an actual implementation knows product state, navigation decisions, or domain rules
- a platform API name appears in a common function name
- adding another target would require changing common callers
- tests need Android or iOS runtime just to verify common business behavior

## Primary Sources

- `https://www.jetbrains.com/help/kotlin-multiplatform-dev/multiplatform-expect-actual.html`
- `https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-multiplatform.html`
- `https://developer.android.com/develop/ui/compose/migrate/multiplatform`
- `https://developer.android.com/develop/ui/compose/interop`
