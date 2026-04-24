# Kotlin Style For Android

The Android Kotlin style guide is a superset of the Kotlin coding conventions — both apply. This file is the subset LLMs most reliably get wrong.

Do not chase style repo-wide when touching a single file. Fix what you write; leave neighboring code alone unless asked.

## Visibility

**Android style defaults to explicit, narrow visibility.** `internal` is the right default for classes that only make sense inside their module. `private` is the right default for helpers only used inside their file.

```kotlin
// Wrong — public by default, leaks an implementation helper
fun formatPrice(price: BigDecimal, locale: Locale): String = ...

// Right — file-private
private fun formatPrice(price: BigDecimal, locale: Locale): String = ...
```

Rule: if the name is not already referenced outside this file or module, it should not be `public`.

## Naming

- Classes: `PascalCase`.
- Functions, properties, parameters, local variables: `camelCase`.
- Top-level `const val`: `SCREAMING_SNAKE_CASE`. Never use it for regular `val` — reserve `SCREAMING_SNAKE_CASE` for true compile-time constants.
- Test methods: `backtick-quoted sentences` are idiomatic (`fun \`returns empty when input is blank\`()`). Android style permits this; plain `camelCase` is also fine. Do not mix both styles in the same module.
- Composables: `PascalCase` noun phrases (see `references/component-api.md`).
- Enum values: `SCREAMING_SNAKE_CASE`. Object values used as enum-ish: `PascalCase`.

## Immutability

- `val` everywhere `val` works. Reach for `var` only when you are genuinely reassigning.
- Prefer `listOf(...)`, `mapOf(...)`, `setOf(...)` (read-only) over `mutableListOf(...)` unless you need to mutate.
- Data classes whose fields are all `val` are implicitly immutable (modulo shared mutable state in their fields). Prefer them for state shapes.

## Data Classes Vs Plain Classes

- Use `data class` when the class's **identity is its data** — `equals`, `hashCode`, and `copy` are desired.
- Do **not** use `data class` for:
  - Classes with mutable state that should have reference equality.
  - Classes that inherit from an abstract class with state.
  - State holders whose identity matters (see `references/state.md` — state holders usually are **not** data classes).

## Sealed Classes And Interfaces

- Use `sealed interface` over `sealed class` for type hierarchies that represent state or result variants. `sealed interface` permits implementation by multiple ancestors (useful for cross-cutting markers like `Loadable` + `Selectable`).
- Exhaustive `when` over a sealed type **should not have an `else` branch**. An exhaustive `when` forces the compiler to flag missing arms when you add a new variant.

```kotlin
sealed interface LoadState<out T> {
    data object Loading : LoadState<Nothing>
    data class Loaded<T>(val value: T) : LoadState<T>
    data class Error(val cause: Throwable) : LoadState<Nothing>
}

when (state) {
    LoadState.Loading -> ...
    is LoadState.Loaded -> ...
    is LoadState.Error -> ...
    // no else — compiler enforces exhaustiveness
}
```

## Nullability

- Do not use `!!`. It is a concession, not an idiom. If you have `!!`, the types or the control flow are wrong.
- `?.let { ... }` is fine for a single optional action. For multiple operations on the same nullable, prefer an early return:

```kotlin
// Weak
fun render(user: User?) {
    user?.let { u ->
        u.render()
        u.track()
    }
}

// Stronger
fun render(user: User?) {
    user ?: return
    user.render()
    user.track()
}
```

- `?:` returns a value; `?: return` / `?: throw` short-circuit. Do not combine both in one expression if it hurts readability.

## Scope Functions

Use with intent. Each of `let`, `run`, `with`, `apply`, `also` has a specific shape. A good reference is the Kotlin docs, but the LLM-common mistakes are:

- `let` used as a sugar for `!!`. `foo?.let { bar(it) }` is fine; `foo.let { bar(it) }` is pointless.
- `apply` for object construction. Use it for configuration of a fresh instance — not for chaining unrelated calls.
- Do not nest scope functions deeply. `.let { .also { .apply { ... } } }` is a code smell.

## Extensions

- Use extension functions for APIs that logically belong to a type but do not need to live in its source file.
- Do **not** use extensions to add core behavior to third-party types (the "pimp my library" anti-pattern). Prefer a wrapper type.
- Extension **properties** should be side-effect free and cheap. If computing the value requires work, it should be a function.

## Functions

- Single-expression functions use `=`:

```kotlin
fun isEmpty(list: List<*>): Boolean = list.isEmpty()
```

- Do not use `= ` for functions whose body is more than a few operations — explicit `{ ... }` scopes are easier to debug.
- Default argument values go in the declaration, not as overloads. Overloads are for genuine polymorphism.
- Trailing lambdas: always use the trailing-lambda syntax when the last parameter is a function. Not:
  ```kotlin
  items.map({ it.name })
  ```
  But:
  ```kotlin
  items.map { it.name }
  ```

## Control Flow

- `when` over long `if/else if/else` ladders.
- `when` with no subject is idiomatic for complex conditions: `when { x > 0 -> ... ; y > 0 -> ... ; else -> ... }`.
- Avoid deeply nested `if`s. Invert the condition and early-return.

## String Templates

- `"$x"` or `"${obj.field}"` over `String.format` for routine interpolation.
- Use `String.format` only when you need locale-aware formatting (numbers, dates, currency). Compose already provides `stringResource(R.string.x, arg)` for localized strings.

## Imports

- No wildcard imports. Android style forbids them.
- Do not alias standard imports (`as FooBar`) unless there is a genuine collision.

## Annotations

- One annotation per line on function / class declarations.
- Application-level annotations that take parameters on the same line as the target are allowed for property accessors and parameters: `@field:Inject val x: Foo`.

## Exceptions

- Do not catch `Exception` or `Throwable` generically in application code. Catch the specific types you can handle.
- Do not rethrow as a different type unless the new type carries strictly more information.
- `runCatching { ... }.getOrElse { ... }` is clean for one-shot error-to-value translation. Do not abuse it to swallow cancellation — cancellation exceptions must propagate in coroutines.

## Formatting

- 4-space indent, no tabs.
- 120-column target where the team agrees; some modules enforce 100. Match the neighborhood.
- Trailing commas on multi-line parameter lists / argument lists / collection literals. They reduce diff noise when adding items.
- One statement per line.
- Imports and package in standard order: package → imports (sorted) → one blank line → file content.

## Android-Specific

- Prefer `Context.getString(R.string.x)` over hard-coded strings in any code path that can be UI-visible. Compose: `stringResource(R.string.x)`.
- Avoid `Parcelize` on anything that is not actually going through a Parcel. It adds bytecode without benefit.
- Resources (`R.id.x`, `R.string.x`) are read-only integers — use them directly, do not wrap in utility accessors.
- `Logger` / `Log` calls should use named tags via a constant (`private const val TAG = "MyClass"`), not hard-coded strings at every call.

## Common LLM-Generated Kotlin Smells

- `object Foo { private val instance = ... ; fun get() = instance }` — that is just a singleton. Drop the layer.
- `companion object { private val TAG = "Foo" }` for a `TAG` that is only used in `Log.d`. A file-level `private const val TAG` is lighter.
- `if (condition) { true } else { false }` — returning a `Boolean` from an `if`. Just write the expression.
- `list.size == 0` — use `list.isEmpty()`.
- `list.size > 0` — use `list.isNotEmpty()`.
- Wrapping idiomatic Kotlin in Java-style builders (`StringBuilder().append(...).toString()`) when `buildString { ... }` is cleaner.

## Primary Sources

- `https://kotlinlang.org/docs/coding-conventions.html` — Kotlin coding conventions.
- `https://developer.android.com/kotlin/style-guide` — Android Kotlin style guide.
- `https://developer.android.com/kotlin/interop` — Kotlin ↔ Java interop rules for public APIs.
