# Compose Skill Suite 4.1.0 Evals

Released: 2026-06-14

These evals are the focused acceptance set for 4.1.0. They cover the Compose API, effect, cancellation, and state-boundary checks that drove the release.

Use them as acceptance criteria for `compose-agent` reviews and `jetpack-compose-audit` scoring/search behavior. They are not a full harness yet; they document the behavior the skills must preserve.

## Eval Cases

### 1. Composable API hygiene

Prompt:

```text
Audit these Compose snippets for API hygiene:
@Composable fun userRow(name: String) { Text(name) }
@Composable fun ActionIcon(m: Modifier = Modifier, enabled: Boolean = true) { Icon(..., modifier = m); Text("label", modifier = m) }
@Composable fun TagList(tags: List<String>, modifier: Modifier = Modifier) { Row { tags.forEach { Text(it) } } }
Return top findings only.
```

Expected behavior:

- Flag Unit-returning UI composables named in lowerCamelCase.
- Flag missing caller-controlled `modifier` on reusable UI.
- Flag the main modifier named `m` instead of `modifier`.
- Flag reusing the caller modifier on multiple emitted children.
- Treat `List<T>` as a contextual stability/hot-path lead, not an automatic severe bug.

### 2. `MutableState<T>` in reusable component APIs

Prompt:

```text
Audit this component contract:
@Composable fun ApiKeyEditor(text: MutableState<String>, isSaving: Boolean, onSaved: () -> Unit, modifier: Modifier = Modifier)
The body mutates text.value directly and calls onSaved from its Save button; the caller's onSaved writes settings.
```

Expected behavior:

- Flag `MutableState<String>` as a reusable composable parameter smell.
- Recommend `text: String` plus `onTextChange: (String) -> Unit`, or a domain-specific state holder.
- Do not flag `onSaved` itself when it is an explicit caller/ViewModel boundary.
- Frame the issue as state ownership and UDF, not generic style.

### 3. Stale callbacks and eager `rememberUpdatedState` reads

Prompt:

```text
Audit this effect code:
@Composable fun LoginBanner(onTimeout: () -> Unit, accountId: String) {
    val latest by rememberUpdatedState(onTimeout)
    val action = remember { val captured = latest; Runnable { captured() } }
    LaunchedEffect(Unit) { delay(5000); onTimeout() }
}
```

Expected behavior:

- Flag `LaunchedEffect(Unit)` calling `onTimeout()` directly when the callback can change.
- Flag `val captured = latest` inside `remember {}` as an eager read that captures the initial callback.
- Recommend either keying the effect on the callback when restart is desired, or reading a `rememberUpdatedState` value at invocation time when restart is not desired.

### 4. UI-emitting composable that returns a value

Prompt:

```text
Audit this reusable composable:
@Composable fun ProviderStatus(provider: Provider): StatusHandle {
    Text(provider.name)
    Icon(...)
    return remember(provider.id) { StatusHandle(provider.id) }
}
```

Expected behavior:

- Flag that a composable should either emit UI or return a value, not both.
- Flag multiple sibling emitters without a cohesive root layout or explicit parent scope.
- Recommend callbacks/state holders instead of returning a handle from content emission.

### 5. Modifier default and forwarding contract

Prompt:

```text
Audit this modifier API:
@Composable fun ProviderCard(provider: Provider, modifier: Modifier = Modifier.padding(8.dp)) {
    Column(Modifier.border(1.dp, Color.Red)) {
        Text(provider.name, modifier)
        Button(modifier = modifier.clickable { }) { Text("Open") }
    }
}
```

Expected behavior:

- Flag `modifier` default not equal to exactly `Modifier`.
- Flag that the caller modifier is not applied to the outermost layout.
- Flag caller modifier reuse on internal child nodes.
- Suggest chaining defaults after the caller modifier on the root and using fresh child modifiers.

### 6. Long UI file shape without inventing boundary bugs

Prompt:

```text
Audit this UI file shape: one public @Composable fun SettingsPanel(...) has 180 lines, five nested Columns/Rows in the main branch, a private helper with KDoc explaining what it does, and repeated string literal test tags. The UI otherwise uses callbacks and a presenter/ViewModel state.
```

Expected behavior:

- Do not invent a high-severity architecture bug when state boundaries are healthy.
- Flag large/deeply nested composables as maintainability and recomposition-reasoning risk.
- Recommend concrete extraction boundaries: toolbar, row, footer, empty state, dialog, repeated section.
- Treat KDoc on a private helper as a naming/scope smell when it compensates for unclear code.
- Treat repeated test-tag strings as low-priority constants.

### 7. Hot lazy row projection work

Prompt:

```text
Audit this hot lazy row model path: every row composable receives items: List<RowItem>, builds Regex(query) in the body, then does items.map { ... }.filter { ... }.joinToString() before rendering. The project has no stability config.
```

Expected behavior:

- Treat `List<RowItem>` as a stability lead that matters more because the path is hot and no stability config exists.
- Flag `Regex(query)` construction in composition.
- Flag chained collection transforms and joining in composition.
- Recommend ViewModel/state-holder projection, or correctly keyed `remember` / `derivedStateOf` only for UI-local cheap derivations.
- Do not claim measured jank without measurement.

### 8. Cancellation swallowing in UI-triggered suspend paths

Prompt:

```text
Audit this suspending UI callback path:
onClick = { scope.launch { try { repository.save(); delay(1000) } catch (t: Throwable) { showError(t) } } }
where scope is GlobalScope and the code catches all failures as UI errors.
```

Expected behavior:

- Flag `GlobalScope` in UI-triggered work.
- Flag `catch (Throwable)` / broad catches around suspend calls as cancellation-swallowing risk.
- Require `CancellationException` to be rethrown before converting failures to UI state.
- Route durable save work to a ViewModel/state holder or caller-owned suspending callback.

### 9. CompositionLocal service dependency leak

Prompt:

```text
Audit these CompositionLocal usages:
val LocalProviderClient = staticCompositionLocalOf<ProviderClient> { error("missing") }
leaf rows read it directly to perform login/logout.
Another LocalSpacing is used for density-independent spacing tokens.
```

Expected behavior:

- Allow ambient UI tokens such as spacing, typography, content color, or density-like environment values.
- Flag service/client `CompositionLocal` dependencies in reusable leaf UI.
- Recommend explicit state/callbacks or ViewModel methods instead of reading clients from leaves.

### 10. Slot ordering and slot lifecycle

Prompt:

```text
Audit this slot API:
@Composable fun Shell(content: @Composable () -> Unit, onClose: () -> Unit, modifier: Modifier = Modifier)
The body calls if (expanded) content() else content().
```

Expected behavior:

- Flag that the main content slot should be the trailing lambda.
- Flag event callback placement so `onClose` is not confused with content.
- Discuss calling the same content slot from multiple branches as a lifecycle/state-preservation risk.
- Recommend a layout shape that preserves slot lifecycle, or `movableContentOf` with a freshness strategy when moving content is intentional.
