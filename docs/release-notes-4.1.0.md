# Compose Skill Suite 4.1.0 Release Notes

Released: 2026-06-14

Compose Skill Suite 4.1.0 is an eval-driven hygiene release. The point is not a new broad category; it is turning concrete review failures into durable skill behavior across both `compose-agent` and `jetpack-compose-audit`.

The release is anchored by [`docs/evals-4.1.0.md`](./evals-4.1.0.md). Those scenarios are the acceptance criteria for the new guidance: if a future edit regresses callback shape, slot lifecycle, stale callback handling, cancellation propagation, or composable API contracts, the release has failed.

## What Changed

### `compose-agent` 4.1.0

- Tightened `references/component-api.md` around composable API shape:
  - event callbacks are not content slots;
  - callback names should read as events (`onClick`, `onSubmit`, `onValueChange`);
  - moved/reused slots need deliberate lifecycle handling;
  - UI-emitting composables should not also return handles or computed values;
  - reusable content composables should emit a cohesive root or declare an explicit parent scope;
  - pure helpers should not be marked `@Composable`;
  - `@ReadOnlyComposable` is for value-only composition reads.
- Tightened `references/effects.md` around stale callback capture:
  - `rememberUpdatedState` values must be read at invocation time when the remembered object should stay stable;
  - eager reads inside `remember { ... }` capture the initial callback.
- Tightened `references/concurrency.md` around cancellation:
  - `CancellationException` must propagate;
  - `runCatching`, `catch (Exception)`, and `catch (Throwable)` around suspend calls need an explicit cancellation guard.

### `jetpack-compose-audit` 4.1.0

- Added search leads for the new eval-covered patterns:
  - `runCatching`, broad catches, and missing `CancellationException` handling;
  - non-`Unit` composable signatures;
  - `@ReadOnlyComposable` misuse;
  - `movableContentOf` slot freshness;
  - callback/slot ordering.
- Added scoring guidance for:
  - stale callback capture through eager `rememberUpdatedState` reads;
  - cancellation swallowing in UI / ViewModel suspend paths;
  - UI-emitting composables that return values;
  - multiple sibling roots without explicit parent scopes;
  - pure helpers incorrectly marked `@Composable`;
  - callback placement and naming issues.
- Added Kotlin coroutine cancellation best practices to the canonical sources so audit deductions can cite an official URL.

## Eval Coverage

The 4.1.0 eval set covers ten focused review cases:

1. composable naming, modifier forwarding, and `List<T>` stability leads;
2. `MutableState<T>` in reusable component APIs;
3. stale callbacks and eager `rememberUpdatedState` reads;
4. UI-emitting composables that return handles;
5. modifier default/forwarding contract;
6. long composable shape without inventing boundary bugs;
7. hot lazy-row projection work in composition;
8. cancellation swallowing in UI-triggered suspend paths;
9. service dependencies hidden behind `CompositionLocal`;
10. slot ordering and slot lifecycle.

## Upgrade

Install or refresh both skills:

```bash
npx --yes skills add hamen/compose_skill --skill '*' -y
```

Claude Code plugin install:

```text
/plugin add hamen/compose_skill --subdir skills/jetpack-compose-audit
/plugin add hamen/compose_skill --subdir skills/compose-agent
```

## Notes

- This is not an install-path breaking release.
- The canonical `skills/<name>/SKILL.md` layout remains unchanged.
- The release also ignores local `tmp/` scratch files so exchanged local artifacts do not pollute `git status`.
