# Animation reference — working notes

Status: **ready for 4.0.0 (compose-agent 4.0.0, jetpack-compose-audit 4.0.0), release materials added.**
Last updated: 2026-06-12.

This is a working doc, not user-facing polish. It records *why* the animation reference exists, where it came from, what we deliberately left out, and the questions still on the table. Keep it honest — it's the place to disagree with past decisions.

## Where this came from

We mine the [`android/skills`](https://github.com/android/skills) issue tracker for feature requests Google leaves to model knowledge plus docs, then implement the sharp parts here. The bet: if developers ask Google for X, Google says "SOTA models already do X / read the docs," and we ship a focused skill for X anyway, we're ahead of what generic docs produce — which is good positioning and good product quality.

Animation came up twice and was left unimplemented upstream:

- **[#8](https://github.com/android/skills/issues/8)** — "Create skill for animation libraries." Closed with the standard "models are good at animations, query the docs" response. A commenter linked their own [`Meet-Miyani/compose-skill`](https://github.com/Meet-Miyani/compose-skill) as a stopgap.
- **[#11](https://github.com/android/skills/issues/11)** — "More Features Requested: Animation, Paging, Performance, UI." Closed, same reasoning, same external repo linked.

We verified the gap is real inside `compose-agent`: animation APIs appeared only as cross-cutting mentions in `api.md`, `effects.md`, and `performance.md` — no systematic API-choice table, no animation-specific anti-pattern list, and no focused scoped-review entry. The audit skill already scored concrete animation defects under existing categories; this work makes the same knowledge usable while writing or reviewing feature-scope code. The `Meet-Miyani` repo confirmed the API surface we'd need to cover but reads as a generic reference; we wrote our own around the *LLM-mistake* thesis instead of copying.

## The thesis (why this isn't a docs reprint)

`compose-agent` only earns its keep by targeting errors models *actually* make. The animation reference is built around tells, not API tours. The four that justify the file:

1. **Wrong API for the job** — models default to `Animatable` + `LaunchedEffect` for things `animate*AsState` does in one line.
2. **`scope.launch` in composition** to react to state — should be `LaunchedEffect(target)` or `animate*AsState`.
3. **`tween(<magic number>)` everywhere** instead of interruptible `spring()`.
4. **Animated reads in composition** that recompose every frame instead of deferring to layout/draw.

(1) and (2) are effect bugs, (4) is a performance bug. That cross-reference into `effects.md` / `performance.md` / `state.md` is the differentiator — a standalone animation cheat-sheet can't make those connections because it doesn't own those other references.

## What we shipped (4.0.0)

- `skills/compose-agent/references/animation.md` — decision table, `updateTransition`, `remember`ed `Animatable`, target-driven launches, gesture `snapTo`/`animateDecay`, reduced motion via `MotionDurationScale`, spec selection, deferred reads, `AnimatedContent`/`AnimatedVisibility`/`animateEnterExit`, `animateItem()`, off-screen infinite transitions, anti-pattern checklist, primary sources.
- `SKILL.md` — new review step #13 (animation), new core instruction, references-list entry, `argument-hint` gains `animation`, description gains "animations."
- Both skill manifest sets → `4.0.0`, `animation` keyword added.
- `jetpack-compose-audit` report instructions → `4.0.0`, with an explicit Performance-side **Animation performance signals** block and chat-summary guidance when animation defects affect the Performance score.
- README headline → 4.0.0, changelog entry.
- Launch materials → `docs/release-notes-4.0.0.md` and `docs/tweet-4.0.0.md`.

## Deliberately out of scope (for now)

- **Separate audit category.** `jetpack-compose-audit` does **not** get a standalone Animation score. Concrete animation defects still count under the existing categories when they are really Performance (`Modifier.offset(x)` per-frame reads), Side Effects (`animateTo()` launched from composition), or Composable API Quality (hard-coded timing in reusable animated components). This keeps the score model stable while still penalizing real bugs.
- **MotionLayout / `ConstraintLayout` animation** (#8 mentioned it). Niche in greenfield Compose; skipped. Revisit only if requested.
- **Lottie / third-party animation libs.** Out of scope by the same rule that keeps Accompanist out — we don't pull in third-party deps without asking.
- **Shared-element transitions** (`SharedTransitionLayout`). Real and increasingly common, but a big enough subsurface to deserve its own decision: fold into `animation.md` or its own file? Parked — see open question 2.

## Open questions (discussion)

1. **Should the audit ever add a standalone Animation category?** Leaning no. The current rubric already scores concrete defects in the existing categories, and adding a fifth category would make historical scores harder to compare.
2. **Shared-element transitions** — own reference file, or a section in `animation.md`? It pulls in `SharedTransitionScope`, `AnimatedVisibilityScope`, `Modifier.sharedElement`, and has its own LLM tells (forgetting the scopes, key mismatches). Probably earns its own file once we see it requested.
3. **Next steal from the issue tracker.** Ranked from the same mining pass:
   - **Paging 3 in Compose** (#11, #25) — **shipped in 4.2.0** as `references/paging.md`; see `docs/paging-skill-plan.md` and convergence testing plan in `docs/release-notes-4.2.0.md`.
   - **(declined by us)** Architecture per-layer (#9), XML→Compose (#38) — both intentionally dropped: DAC covers architecture, and XML is dead weight in 2026.
4. **Provenance signaling.** Worth being louder (README? per-reference footer?) that these references are sourced from declined upstream requests? Good marketing story, but don't want it to read as adversarial toward the Android team. Current choice: a single neutral line in the changelog. Revisit.

## Validation

- [x] Ran a fixture smoke-check against deliberate animation bugs (`Animatable` not remembered, target-driven `scope.launch`, `tween(300)` on interactive state, `Modifier.offset(x)` / `Modifier.alpha(a)` composition reads, `AnimatedContent` without `contentKey`, missing lazy-list keys for `animateItem()`).
- [x] Sanity-checked every `developer.android.com` link in `animation.md` resolves (`choose-api`, `value-based`, `customize`, `composables-modifiers`, performance deferred reads, lazy-list item animations, `MotionDurationScale`).
- [x] Confirmed `animation.md` no longer contradicts `effects.md`, `performance.md`, or `modifiers.md` on event-driven `rememberCoroutineScope()` and lambda-form modifier guidance.
