# Compose Skill Suite 4.0.0 Release Notes

Released: 2026-06-12

Compose Skill Suite 4.0.0 makes Jetpack Compose animation a first-class review and authoring surface for AI coding agents.

The release started from the Android `skills` issue tracker. Animation support was requested in [`android/skills#8`](https://github.com/android/skills/issues/8) and again in [`android/skills#11`](https://github.com/android/skills/issues/11), but upstream left it to general model knowledge plus the Android docs. We took the opposite product bet: animation is exactly where agent skills earn their keep, because models repeatedly make the same subtle mistakes.

## What Changed

### `compose-agent` 4.0.0

- Added `skills/compose-agent/references/animation.md`.
- Added an explicit animation review step to `SKILL.md`.
- Added an authoring-mode guardrail: if code animates, the agent checks API choice, remembered animation state, lifecycle, labels, and phase-correct reads before returning code.
- Added animation discovery terms to the Claude and Cursor manifests.

The new animation reference is organized around LLM tells:

- Using `Animatable` when `animate*AsState` or `updateTransition` is enough.
- Creating `Animatable(...)` in composition without `remember`.
- Launching target-driven animations from `rememberCoroutineScope().launch` instead of `LaunchedEffect(target)`.
- Applying `tween(300)` to everything instead of using interruptible `spring()` motion where appropriate.
- Reading animated values in composition so the whole subtree recomposes every frame.
- Forgetting `AnimatedContent` `contentKey`, lazy-list `animateItem()` keys, or `AnimatedVisibility` child `animateEnterExit`.
- Keeping infinite decorative motion composed when the content is offscreen.

### `jetpack-compose-audit` 4.0.0

- Keeps the existing four scored categories: Performance, State management, Side effects, and Composable API quality.
- Does not add a separate Animation score. Animation defects are scored where the root cause belongs.
- Adds an explicit **Animation performance signals** block to the Performance report template.
- Adds an explicit **Animation side-effect signals** block to the Side Effects report template.
- Updates scoring/search guidance so animation findings are named as **Animation phase correctness** instead of being buried under generic recomposition language.
- Updates the final chat summary guidance so animation-related score changes are visible in the short result, not just the full report.

## What Users Can Prove

This release gives users concrete, source-backed findings instead of animation style advice:

- A drawer, carousel, sheet, or animated list can stop rebuilding the surrounding composable tree every frame when animated reads move to layout/draw phase.
- An `Animatable` can stop resetting on recomposition once remembered or hoisted.
- A target-driven animation can get correct restart/cancellation semantics by moving from ad hoc `scope.launch` to `LaunchedEffect(target)`.
- Multi-property transitions can stop drifting when several `animate*AsState` calls driven by the same enum/boolean become one `updateTransition`.
- Offscreen infinite transitions can stop doing continuous work when scoped to visible content.

Every rule is tied back to official Android Developers documentation or AndroidX reference docs.

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

- This is a major release because animation is now a public, documented, and tested skill surface across both skills.
- It is not an install-path breaking release. The canonical `skills/<name>/SKILL.md` layout from 3.0.0 remains unchanged.
- Third-party animation libraries, MotionLayout, and shared-element transitions remain out of scope for now.
