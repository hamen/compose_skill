# Compose Skill Suite 4.1.0 Tweet

## Primary Tweet

Compose Skill Suite 4.1 is out.

This is an eval-driven hygiene release: tighter Jetpack Compose API contracts, slot/callback checks, stale callback detection, cancellation-safe suspend paths, and sharper audit scoring.

The eval set is public in the repo.

https://github.com/hamen/compose_skill

## Shorter Variant

Compose Skill Suite 4.1 is out.

Eval-driven Compose review hygiene:

- callback vs content slot shape
- modifier forwarding
- stale rememberUpdatedState captures
- CancellationException propagation
- emit-UI-and-return-value composables

https://github.com/hamen/compose_skill

## Optional Thread

1. Compose Skill Suite 4.1 is out. This is a focused, eval-driven release for the Compose review mistakes agents still miss after the big animation update.

2. The main artifact is the eval set: ten concrete cases covering modifier contracts, MutableState APIs, stale callbacks, slot lifecycle, cancellation swallowing, CompositionLocal service leaks, and hot-path projection work.

3. `compose-agent` now carries those rules in `component-api.md`, `effects.md`, and `concurrency.md`, so authoring and review mode both see them.

4. `jetpack-compose-audit` now searches and scores the same patterns, with official citation paths for the new coroutine cancellation checks.

5. Install both skills:

`npx --yes skills add hamen/compose_skill --skill '*' -y`

https://github.com/hamen/compose_skill
