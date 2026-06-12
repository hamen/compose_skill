# Compose Skill Suite 4.0.0 Tweet

## Primary Tweet

Compose Skill Suite 4.0 is out.

Jetpack Compose animation is now a first-class review surface for AI coding agents: API choice, remembered Animatable, LaunchedEffect targets, updateTransition, reduced motion, and phase-correct reads.

https://github.com/hamen/compose_skill

## Shorter Variant

Compose Skill Suite 4.0 is out.

First-class Jetpack Compose animation guidance for AI coding agents.

It catches what LLMs keep getting wrong: wrong API choice, unremembered Animatable, scope.launch for target state, tween-everything, and per-frame recomposition.

## Optional Thread

1. Compose Skill Suite 4.0 is out. This release makes Jetpack Compose animation a first-class surface across both skills: write/review with `compose-agent`, then audit real repos with `jetpack-compose-audit`.

2. The idea came from Android's public `skills` issue tracker. Animation support was requested in issues #8 and #11, then left to model knowledge plus docs. We turned that gap into a focused skill surface.

3. The new `animation.md` reference is built around LLM tells: wrong API choice, `Animatable` without `remember`, target-driven `scope.launch`, `tween` everywhere, and animated reads that recompose every frame.

4. The audit does not add a vanity Animation score. Real animation defects now land where they belong: Performance, Side Effects, or Composable API Quality, with explicit animation blocks in the report.

5. Install both skills:

`npx --yes skills add hamen/compose_skill --skill '*' -y`

https://github.com/hamen/compose_skill
