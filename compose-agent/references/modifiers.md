# Modifiers

Modifier rules LLMs routinely break. Most of them are encoded in the AndroidX component guidelines, but the summary below is what actually matters in code review.

## The Modifier Parameter Contract

Every composable that renders layout-affecting UI should accept a `modifier` parameter.

```kotlin
@Composable
fun UserCard(
    user: User,
    onClick: () -> Unit,
    modifier: Modifier = Modifier, // typed Modifier, defaulted, named 'modifier'
) {
    Card(
        modifier = modifier // applied to the outermost layout
            .clickable(onClick = onClick),
        // ...
    ) { /* content */ }
}
```

Rules:

1. **Exactly one** `modifier` parameter per composable.
2. Parameter is **named** `modifier`, typed `Modifier`, defaulted `= Modifier`.
3. Applied to the **outermost** layout-producing call — so the caller's size, padding, background, and click modifiers work.
4. Any internal modifiers are chained **after** `modifier`, not before. The caller's instructions come first, so internal ones (like a semantic clickable or a required fill) can override or refine.
5. Never create a `modifier` you do not forward. If your composable genuinely has no outer layout (e.g. it only decides what to render and returns another composable), skip the parameter entirely.

Violations are easy to grep for:

- A composable that takes `modifier: Modifier = Modifier` but never references `modifier` in its body.
- A composable that applies `modifier` to an **inner** element (e.g. a child `Text`) instead of the outer `Column`/`Box`/`Row`.

## Modifier Order Matters

Modifiers are a chain, applied outer-in. The order changes behavior.

```kotlin
// Wrong — padding is outside the background, so the background doesn't cover the padded region
Box(
    modifier = Modifier
        .padding(16.dp)
        .background(Color.Red)
        .size(100.dp)
)

// Right — background covers the whole sized area, padding is applied inside (shrinks content area)
Box(
    modifier = Modifier
        .size(100.dp)
        .background(Color.Red)
        .padding(16.dp)
)
```

Helpful ordering mnemonic for a single element:

1. Size / fill / weight (outer bound of the element).
2. Shape / background / border (visual box).
3. Click / semantics (interaction).
4. Padding (content inset).

LLMs frequently invert 1 and 2, or put `clickable` after `padding` so the padding itself is not clickable. Flag.

## Lambda-Form Modifiers For Frequently Changing Values

See `references/performance.md`. Short version:

- `Modifier.offset(x = xDp)` → reads in composition. Use `Modifier.offset { IntOffset(...) }` for animated / scroll-driven values.
- `Modifier.padding(horizontal = dynamicDp)` → same rule, use `Modifier.padding { ... }` in the rare cases where you are animating padding.
- `Modifier.alpha(a)` → use `Modifier.graphicsLayer { this.alpha = a }` when `a` is animated.
- `Modifier.rotate(r)` → `Modifier.graphicsLayer { rotationZ = r }`.
- `Modifier.scale(s)` → `Modifier.graphicsLayer { scaleX = s; scaleY = s }`.

`Modifier.graphicsLayer { ... }` is the generic escape hatch for per-frame visual values. It also gives you a free rasterization layer, so do not sprinkle it everywhere — only on elements you are animating.

## Custom Modifiers — `Modifier.Node`, Not `composed { }`

`Modifier.composed { ... }` is **discouraged** and slower than the node API. Every instance allocates composition-local state and opens a new composable scope.

```kotlin
// Discouraged
fun Modifier.rainbowBorder(): Modifier = composed {
    val color by rememberRainbow()
    this.border(2.dp, color)
}

// Preferred — Modifier.Node
class RainbowBorderElement : ModifierNodeElement<RainbowBorderNode>() {
    override fun create() = RainbowBorderNode()
    override fun update(node: RainbowBorderNode) { /* reconfigure */ }
    override fun hashCode(): Int = 0
    override fun equals(other: Any?): Boolean = other === this
}

class RainbowBorderNode : Modifier.Node(), DrawModifierNode {
    override fun ContentDrawScope.draw() {
        // draw rainbow border, then drawContent()
    }
}

fun Modifier.rainbowBorder(): Modifier = this then RainbowBorderElement()
```

Rules:

- For new custom modifiers, use `Modifier.Node` and its delegates (`DrawModifierNode`, `LayoutModifierNode`, `PointerInputModifierNode`, etc.).
- Use `composed { }` **only** when you need a composable context inside the modifier (e.g. read a `CompositionLocal` that has no non-composable access path). Prefer passing the value into the modifier as a parameter instead.
- Never return a `Modifier.composed { remember { ... } }` where `remember` is only caching an object — allocate in the element class instead.

## Chaining And Reuse

- `Modifier` is a value. Instances can be stored in a `val`, but **do not** `remember` a `Modifier` unless you are convinced it is an expensive chain that is identical across recompositions. In almost all cases Compose handles this better than manual caching.
- Build modifiers inline at the call site. Named module-level modifier constants are fine for truly shared chains (spacing constants, standard card chrome).
- `then` and the overloaded operator `Modifier.thenMaybe { }` (in some utility libraries) are both fine, but prefer readable chains over clever combinators.

## Modifier Factories As Extension Functions

Custom modifier APIs should be extension functions on `Modifier`:

```kotlin
// Right
fun Modifier.highlighted(color: Color): Modifier = this.background(color).border(1.dp, color.darker())

// Wrong — does not chain, breaks the modifier fluency
fun highlighted(modifier: Modifier, color: Color): Modifier = modifier.background(color)...
```

Rule: if it takes a `Modifier` in and returns a `Modifier`, it must be `Modifier.foo(...)`, not `foo(modifier, ...)`.

## Semantic Modifiers

- `Modifier.clickable(onClick = ...)` — default role is a button. If the tap target is not a button, pass `role = Role.Switch` / `Role.Checkbox` / etc.
- `Modifier.toggleable`, `Modifier.selectable`, `Modifier.triStateToggleable` — use these in preference to `clickable` when the interaction has a state.
- For icon-only touch targets that trigger an action, always pair the modifier with a content description through either `contentDescription` on an underlying `Icon` or `Modifier.semantics { contentDescription = ... }`. Icon-only buttons without labels are the most common accessibility hole Compose agents produce.

## `Modifier.pointerInput` Keys

```kotlin
// Wrong — recreates the gesture detector on every recomposition, losing in-flight state
Modifier.pointerInput(Unit) { detectDragGestures(...) } // ok, stable key
Modifier.pointerInput(someChangingValue) { detectDragGestures(...) } // restarts gesture every change
```

The keys passed to `pointerInput` restart the suspend block on change. Pass stable keys — usually `Unit`, or `key =` that truly should reset the gesture handling.

## Grep Triggers

- composables taking `modifier: Modifier = Modifier` with no `modifier` reference in the body
- `Modifier\.composed\s*\{` — candidates for `Modifier.Node`
- `Modifier\.(offset|alpha|scale|rotate)\s*\(` with an animated `.value` — use lambda form
- `Modifier\.pointerInput\s*\(\s*\w+` where the key is not `Unit` — verify it should reset on change
- icon-only `Button`, `IconButton`, or `clickable`-applied `Icon` without a `contentDescription` or `Modifier.semantics`

## Primary Sources

- `https://developer.android.com/develop/ui/compose/modifiers`
- `https://developer.android.com/develop/ui/compose/custom-modifiers`
- `https://developer.android.com/develop/ui/compose/graphics/draw/modifiers`
- AndroidX component API guidelines — `compose-component-api-guidelines.md`
