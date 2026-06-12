<h1 align="center">Compose Skill Suite</h1>

<p align="center">
  <strong>Jetpack Compose audit + coding-agent skills for Claude Code, Codex, Cursor, and Anthropic-style skill loaders.</strong>
</p>

<p align="center">
  <a href="./bin/ci"><img alt="bin/ci passing" src="https://img.shields.io/badge/bin%2Fci-passing-2ea043"></a>
  <a href="https://github.com/hamen/compose_skill/releases/tag/v4.0.0"><img alt="Release" src="https://img.shields.io/github/v/release/hamen/compose_skill?color=2f80ed&label=release"></a>
  <a href="LICENSE"><img alt="License" src="https://img.shields.io/github/license/hamen/compose_skill?color=0a7f60"></a>
  <img alt="Skills" src="https://img.shields.io/badge/skills-2-7c3aed">
  <img alt="Compose animation ready" src="https://img.shields.io/badge/Compose-animation%20ready-f97316">
  <img alt="Claude Code plugin" src="https://img.shields.io/badge/Claude%20Code-plugin-111827">
</p>

**Version 4.0.0 · released 2026-06-12** — Major animation release. Compose animation is now a first-class surface across the suite: `compose-agent` gains a dedicated `animation.md` reference targeting the mistakes LLMs make most, while `jetpack-compose-audit` surfaces animation performance and side-effect defects in the scored report. Both skills ship as `4.0.0`.

> Find out where your Compose app is burning frames, by how much, and what to change to win them back — measured against real compiler data, not vibes.

A strict, evidence-based audit for Android Jetpack Compose repositories. Point it at a repo, let it run the build once, and get back a 0-100 score, a 0-10 score per category, an actionable top-three fix list, and a full Markdown report with every deduction cited against an official `developer.android.com` page.

Built for Claude Code, Codex, Cursor, and any agent that loads the Anthropic skill format.

Authored and cross-reviewed with every frontier model — Claude Opus 4.8, GPT-5.5, and Gemini 3.5 Flash — each used to pressure-test the rubric, prompts, and references so the audit holds up regardless of which model ends up running it.

---

## What's new

### 4.0.0 — 2026-06-12

**Major Compose animation release.**

Animation is one of the largest API surfaces in Compose and the one LLMs misuse most predictably. This release makes animation a first-class skill surface — staying true to the suite's thesis of targeting *the mistakes models actually make*, not re-explaining the docs.

- **New `references/animation.md`.** Covers API selection (`animate*AsState` vs `updateTransition` vs `Animatable`), synchronized multi-property motion with `updateTransition`, the `remember`ed-`Animatable` rule, target-driven launches via `LaunchedEffect`, gesture-driven `snapTo`/`animateDecay`, reduced motion, `spring` vs `tween` vs `keyframes`, deferred animated reads (layout/draw phase, `graphicsLayer`), `AnimatedContent` `contentKey` + `transitionSpec`, `AnimatedVisibility` + `animateEnterExit`, `Modifier.animateItem()` for lazy lists, and off-screen infinite transitions. Each section is framed as an **LLM tell** with a before/after fix and an official `developer.android.com` citation.
- **New review step + core instruction.** The `compose-agent` review process now includes an explicit animation pass, and the core instructions carry a one-line "animate declaratively first" guardrail for authoring mode.
- **Origin.** Mined from the [`android/skills`](https://github.com/android/skills) issue tracker — animation coverage was requested (issues #8, #11) and left to model knowledge plus docs upstream, leaving a practical gap this skill now fills. The cross-referenced relationship with `performance.md` (deferred reads) and `effects.md` (target-driven launches) is what differentiates it from a generic animation cheat-sheet.
- **Audit reporting clarity.** `jetpack-compose-audit` has no separate Animation score category, but concrete animation defects still affect the existing Performance, Side Effects, or Composable API Quality scores when they violate those rubrics. The Performance report template now includes an explicit **Animation performance signals** block so these findings do not disappear under generic recomposition wording.
- **Launch materials.** Release notes live in [`docs/release-notes-4.0.0.md`](./docs/release-notes-4.0.0.md); the launch tweet lives in [`docs/tweet-4.0.0.md`](./docs/tweet-4.0.0.md).
- **Versions.** `compose-agent` → `4.0.0`. `jetpack-compose-audit` → `4.0.0`.

### 3.1.0 — 2026-06-08

**Android 12+ splash icon blur detection.**

There's a years-old Android bug ([issuetracker 520672537](https://issuetracker.google.com/issues/520672537)): if your Android 12+ splash icon is a static drawable, the system pre-renders it at 108 dp and scales it up to 160/192 dp — so a crisp vector shows up **blurry** on most phones (XHDPI and above). The fix is to wrap the same vector in an `<animated-vector>` with no animators, which routes it to the full-size render path. This release teaches both skills to catch it and fix it.

- **The audit now flags it automatically.** `jetpack-compose-audit` scans your splash-screen theme resources during a normal audit — no extra command. It resolves `windowSplashScreenAnimatedIcon`, follows the drawable, and reports when API 31+ resolves to a static icon instead of an `<animated-vector>`.
- **It's a non-scored finding, but still surfaced.** Splash icon risk lives under *Android Launch UX*, outside the four numeric Compose categories, so it never moves your 0–100 score — but it still shows up in *Critical Findings* / *Prioritized Fixes* when it's real.
- **The agent knows the fix.** `compose-agent` documents the one-line workaround (an empty `<animated-vector>` wrapping your vector) and the single trap that breaks naive attempts — the `drawable-v31` self-reference loop, where the wrapper must point at a *differently-named* vector.
- **Grounded in the actual AOSP source**, not guesswork: the docs explain why an `AnimatedVectorDrawable` (it's `Animatable`, drawn at full size) dodges the `ImmobileIconDrawable` 108 dp pre-render, plus the official icon sizing spec.
- **Versions.** `jetpack-compose-audit` → `3.1.0`, `compose-agent` → `2.1.0`.

### 3.0.0 — 2026-05-29

**Breaking — direct `skills/<name>/SKILL.md` layout.**

- **Modern skill layout**: `compose-agent` now lives at `skills/compose-agent/SKILL.md`; `jetpack-compose-audit` now lives at `skills/jetpack-compose-audit/SKILL.md`.
- **No compatibility wrappers**: the previous `compose-agent/`, `jetpack-compose-audit/`, and nested `skills/<plugin>/skills/<name>/` layouts are intentionally removed. Keeping one canonical layout keeps this repo clean and gives other Compose-related skills a clear example to follow.
- **Install paths changed**: existing installs do not migrate automatically because agents remember the old subdirectory. Remove the old install once and reinstall from the new path.
- **No behavior change**: audit scoring, references, scripts, report format, and `compose-agent` review rules are unchanged.
- **Versions.** `jetpack-compose-audit` → `3.0.0`. `compose-agent` → `2.0.0`.

Migration:

```bash
# npx skills users
npx --yes skills remove compose-agent jetpack-compose-audit -y
npx --yes skills add hamen/compose_skill --skill '*' -y
```

Claude Code users who installed with the old subdirectories should remove the old plugin install, then reinstall with:

```
/plugin add hamen/compose_skill --subdir skills/jetpack-compose-audit
/plugin add hamen/compose_skill --subdir skills/compose-agent
```

This is a breaking change on purpose: one clean layout is better than carrying repository-shape debt forever.

### 2.1.1 — 2026-05-20

**Fixed — Windows plugin loading.**

- **Claude Code plugins**: both `compose-agent` and `jetpack-compose-audit` now use the documented `skills/<name>/SKILL.md` layout instead of a root-level skill plus `skills: "./"` in the manifest. This avoids Claude Code for Windows rejecting the plugin with `Path escapes plugin directory: ./ (skills)`.
- **Manual installs**: symlink commands now point directly at each nested skill directory for Claude Code, Codex, and Cursor.
- **Versions.** `jetpack-compose-audit` → `2.1.1`. `compose-agent` → `1.2.1`. No scoring, rubric, or report-format changes.

For older releases see the [full Changelog](#changelog) at the bottom of this README.

---

## What you get

Run the skill on a Compose repo and you walk away with:

- **`COMPOSE-AUDIT-REPORT.md`** written at the target root — per-category scoring, evidence file paths, line numbers, and prioritized fixes.
- **A chat summary** that mirrors the report's top three fixes — same file paths, same doc links, same predicted impact. Act on the chat alone if you're short on time.
- **Measured stability numbers** from the Compose Compiler — module-wide `skippable%`, named-only `skippable%`, the unstable-class list, and the per-module Strong Skipping state inferred from compiler version plus explicit flags.
- **Android Launch UX findings** for static Android 12+ splash icons that can render blurry when a `drawable-v31` animated-vector wrapper is missing.
- **A score you can defend.** Every deduction carries an official Android Developers URL. No "trust me" findings.

---

## What the audit looks at

Four categories, weighted for an app repo. Each scored `0-10`; overall on `0-100`.

| Category | Weight | What it covers |
|----------|--------|----------------|
| **Performance** | 35% | Work in composition, lazy-list keys, state-read timing, stability, Strong Skipping, backwards writes, **animation phase correctness**, baseline profiles |
| **State management** | 25% | Hoisting, single source of truth, `rememberSaveable`, lifecycle-aware collection, observable collections, ViewModel placement, type-safe navigation |
| **Side effects** | 20% | Effect API choice, keys, stale captures, cleanup, composition-time work, **animation driving via `LaunchedEffect`** |
| **Composable API quality** | 20% | Modifier conventions, parameter order, slot APIs, `CompositionLocal` usage, `Modifier.Node`, **`animationSpec` exposure**, `@Preview` coverage, hardcoded strings / magic numbers |

Score bands: `0-3` fail · `4-6` needs work · `7-8` solid · `9-10` excellent.

Adjacent, non-scored coverage includes UI tests/previews, focus/keyboard, KMP/CMP, and Android Launch UX resources. Those findings do not change the 0-100 score, but they can still show up in Critical Findings and Prioritized Fixes when they are concrete and user-visible.

---

## What it catches (and what fixing it buys you)

Concrete smells the rubric targets, with realistic wins:

| Smell | Expected gain after fix |
|-------|-------------------------|
| Unstable or repeatedly recreated params (`List`, domain models, `ArrayList`-backed state, `listOf(...)`, fresh UI models) | On older compiler tracks, can lift named-only `skippable%` and the Performance ceiling. Under Strong Skipping, usually removes instance-recreation churn or expensive `equals()` work that was still forcing re-runs despite high skippability. |
| Lazy-list `items(...)` without stable `key =` | Fewer reallocated compositions on reorder, smoother scroll, fewer `IllegalArgumentException: Key already used` crashes |
| Rapidly-changing state read high in the tree | Recompositions collapse from "per frame, whole screen" to "per frame, single modifier" |
| Animated `.value` piped into `Modifier.offset(x.dp)` / `Modifier.alpha(a)` | Moving to `Modifier.graphicsLayer { ... }` / `Modifier.offset { ... }` defers per-frame reads to layout/draw — same animation, fraction of the recomposition cost |
| `Animatable(...)` created in a composable body without `remember` | Animation no longer resets on every recomposition; velocity and target survive |
| `rememberCoroutineScope().launch { animatable.animateTo(...) }` for target-driven animation | Replace with `LaunchedEffect(target)` — restart semantics follow the target automatically, while `rememberCoroutineScope()` stays available for event-driven animation |
| `rememberInfiniteTransition` hosted on something that stays composed offscreen | Scoping it to visible content avoids needless offscreen animation work and lets it stop when the host actually leaves composition |
| `collectAsState()` on Android UI flows | Swap to `collectAsStateWithLifecycle()` — no collection when UI is paused |
| `mutableStateOf<Int>` / `<Long>` / `<Float>` in hot paths | Remove autoboxing, fewer allocations |
| Hardcoded strings and magic numbers in reusable components | i18n + dark-mode + accessibility ready; testable |
| `rememberSaveable` inside a `LazyListScope` item factory | No more `TransactionTooLargeException` when the list grows |
| `Scaffold { innerPadding -> ... }` content that ignores `innerPadding` | Content stops drawing behind the `TopAppBar` / system bars |
| Static Android 12+ splash icon in `windowSplashScreenAnimatedIcon` | Wrap with a `drawable-v31` animated-vector so the launch icon stays crisp instead of being rasterized small and scaled up |

The report lists every occurrence with file path and line number, not just the category.

## What makes it different

**Measured, not inferred.** The skill ships `scripts/compose-reports.init.gradle` and injects it into your Gradle build via `--init-script` — no edits to your `build.gradle`. Every run parses real `*-classes.txt` / `*-composables.txt` / `*-module.json` output.

**Mandatory ceilings.** A Performance score cannot exceed the cap set by the matching ceiling table. On older compiler tracks the cap is driven by `skippable%` plus unstable-param count; under Strong Skipping it is driven by named-only `skippable%`, instance-recreation churn, and `equals()` quality on unstable params. The ceiling math appears in the report so the score is auditable.

**Every deduction cites an official source.** Each finding carries a `References:` line pointing at `developer.android.com` or the AndroidX component API guidelines. Audits that can't be defended with a URL don't ship.

**Actionable chat summary.** The chat output mirrors the report's `Prioritized Fixes` — same file paths, same doc links, same predicted impact ("stops rebuilding `FeedItemUiModel`, removes the Strong-Skipping cap from 8 → no cap").

---

## Install

### Recommended: `npx skills`

Install both skills:

```bash
npx --yes skills add hamen/compose_skill --skill '*' -y
```

Or install one skill:

```bash
npx --yes skills add hamen/compose_skill --skill jetpack-compose-audit -y
npx --yes skills add hamen/compose_skill --skill compose-agent -y
```

This is the preferred path for Codex, Claude Code, Cursor, and multi-agent setups because the repo now follows the direct `skills/<name>/SKILL.md` layout.

### Claude Code plugin install

Direct plugin install still works if you prefer Claude Code's plugin flow:

```
/plugin add hamen/compose_skill --subdir skills/jetpack-compose-audit
```

Claude Code reads `skills/jetpack-compose-audit/.claude-plugin/plugin.json` and registers `skills/jetpack-compose-audit/SKILL.md`. For `compose-agent`, use:

```
/plugin add hamen/compose_skill --subdir skills/compose-agent
```

### Breaking migration

If you installed an older release with `--subdir compose-agent`, `--subdir jetpack-compose-audit`, or the nested `skills/<plugin>/skills/<name>` manual symlink, `/plugin update` will keep pointing at the old path. Remove that old install once and reinstall with the commands above.

### Manual symlink

Use this when you want `git pull` in this directory to update a local checkout in place:

```bash
mkdir -p ~/.claude/skills
mkdir -p ~/.codex/skills
mkdir -p ~/.cursor/skills

ln -s "$(pwd)/skills/jetpack-compose-audit" ~/.claude/skills/jetpack-compose-audit
ln -s "$(pwd)/skills/jetpack-compose-audit" ~/.codex/skills/jetpack-compose-audit
ln -s "$(pwd)/skills/jetpack-compose-audit" ~/.cursor/skills/jetpack-compose-audit

ln -s "$(pwd)/skills/compose-agent" ~/.claude/skills/compose-agent
ln -s "$(pwd)/skills/compose-agent" ~/.codex/skills/compose-agent
ln -s "$(pwd)/skills/compose-agent" ~/.cursor/skills/compose-agent
```

---

## Use

From the agent prompt:

```
/jetpack-compose-audit [repo path or module path]
```

Or in natural language:

```
Audit this Compose repo.
Score the :app module for Compose quality.
Run a Compose performance review on core/ui.
```

The compiler-report build runs automatically and typically takes 1-5 minutes. If the build fails (no wrapper, compile error, timeout) the skill falls back to source-inferred findings, caps Performance at `7`, and flags reduced confidence — all stated explicitly in the report.

---

## Example output

```
Overall: 73/100

Performance:  8/10  capped by the SSM-on table: instance-recreation churn in feed params (qualitative 9)
State:        6/10  collectAsState without lifecycle, duplicate VM reads
Side effects: 7/10  LaunchedEffect key too broad at HomeScreen.kt:240
API quality:  8/10  BoxCard / SearchBar follow conventions

Compiler:
  Strong Skipping: on (default)
  ceiling table: SSM-on
  module-wide skippable% = 186/269 = 69.14%
  named-only skippable% = 121/122 = 99.18%
  ceiling metric: named-only `skippable%` (module-wide metric anchored by zero-arg lambdas)
  deferredUnstableClasses: 59
  binding cap: 8 (fresh `FeedItemUiModel(...)` + `listOf(...)` rebuilt in `HomeFeedScreen`)

Top 3 fixes
1. collectAsState -> collectAsStateWithLifecycle across 6 call sites
   feature/home/HomeScreen.kt:37, MainActivity.kt:213, ...
   Doc: developer.android.com/.../side-effects
   Impact: fewer redundant collections, lifecycle-correct

2. Stop rebuilding `FeedItemUiModel(...)` and `listOf(...)` inside `HomeFeedScreen`
   Evidence: app/build/compose_audit/app_release-classes.txt, feature/home/HomeFeedScreen.kt:88-132
   Doc: developer.android.com/.../stability
   Impact: removes forced re-runs under Strong Skipping, likely clears the Performance cap from 8 -> no cap

3. Narrow LaunchedEffect(homeScreenState) at HomeScreen.kt:240-254
   Doc: developer.android.com/.../side-effects
   Impact: fewer redundant ensureAuthenticated() calls
```

---

## Scope

**In scope.** Jetpack Compose on Android, Kotlin 2.0.20+ / Compose Compiler 1.5.4+ (Strong Skipping default).

Also in normal audit coverage: Android splash-screen launch resources, specifically `windowSplashScreenAnimatedIcon` and `drawable-v31` animated-vector overrides for the Android 12+ static-icon blur workaround. This is reported as Android Launch UX, not as a scored Compose category.

**Out of scope** — the skill will call these out as a note rather than silently produce thin coverage:

- Material 3 compliance, theming, color/typography — defer to the `material-3` skill.
- Accessibility scoring (semantics, touch targets) — flagged as notes, not scored.
- UI test coverage and Compose test-rule patterns — noted as adjacent coverage, not scored.
- Compose Multiplatform (`expect`/`actual`, target-specific code paths) — noted as adjacent coverage, not scored.
- Wear OS / TV / Auto / Glance surfaces — focus/keyboard risks are noted as adjacent coverage; full platform review remains out of scope.
- Build performance (incremental compilation, KSP/KAPT choice).

---

## Layout

```
skills/
  jetpack-compose-audit/
    .claude-plugin/plugin.json     Claude Code plugin manifest
    .cursor-plugin/plugin.json     Cursor plugin manifest
    SKILL.md                       main audit skill (process, principles, output)
    scripts/
      compose-reports.init.gradle  Gradle init script injected via --init-script
    references/
      scoring.md                   rubric with measured ceilings and inline citations
      search-playbook.md           grep patterns, regex, read-the-file heuristics
      canonical-sources.md         every URL the rubric cites
      report-template.md           required structure for COMPOSE-AUDIT-REPORT.md
      diagnostics.md               manual-mode fallback snippets
  compose-agent/                   sibling skill — see § Sibling skill below
```

A sibling skill, `skills/compose-agent/`, ships in the same repo — see [§ Sibling skill](#sibling-skill--compose-agent) for its own layout and usage.

---

## Philosophy

- **Strict but evidence-based.** Every deduction has a file:line and an official-doc URL.
- **Measured beats inferred.** Compiler reports are generated automatically; source-inferred stability is a fallback, not the default.
- **Written for action.** The report's `Prioritized Fixes` section and the chat summary mirror each other, so the developer can act on the chat alone.
- **Narrow scope on purpose.** The skill does not score design, accessibility, or build performance in v1. It says so rather than pretending otherwise.

---

## Sibling skill — `compose-agent`

This repo ships a second skill alongside the audit: [`compose-agent/`](./skills/compose-agent/). Where the audit **reviews an existing repo** end-to-end and produces a score, `compose-agent` works at **file and feature scope** while you are reading, writing, or modifying Compose.

- **Responds to:** "is this right?", "rewrite this the modern way", "check this file for deprecated API", "find state hoisting mistakes in this feature".
- **Built for:** [android/skills#27](https://github.com/android/skills/issues/27) — the Android equivalent of [`swiftui-agent-skill`](https://github.com/twostraws/swiftui-agent-skill). The philosophy is the same: target the mistakes LLMs actually make in Compose, not repeat basics the model already knows.
- **Shape:** short `SKILL.md` that routes to focused per-topic reference markdowns. You only pay the token cost for the areas your current task touches.

### Install

Use the same direct skills flow as the audit skill.

**Recommended:**

```bash
npx --yes skills add hamen/compose_skill --skill compose-agent -y
```

**Claude Code:**

```
/plugin add hamen/compose_skill --subdir skills/compose-agent
```

**Cursor:** import the repo as a plugin and pick `compose-agent` in the subdirectory selector.

**Manual:** symlink `skills/compose-agent/` into your skills directory (`~/.claude/skills/compose-agent`, `~/.cursor/skills/compose-agent`, etc.).

Both skills can live side by side — they do not share state and do not interfere.

### Use it — the two modes

`compose-agent` runs in **review mode** or **authoring mode**. You do not choose; the skill picks based on your request.

**Review mode** — you hand it code that already exists. It produces a file-by-file report with before/after snippets and links to the official doc page behind each rule. Triggered by prompts like:

```
Use compose-agent to review feature/profile/.
Check ProfileScreen.kt with compose-agent.
compose-agent: find deprecated API in this module.
```

**Authoring mode** — the skill is loaded and you ask the assistant to *write* Compose. Before returning code, the assistant silently runs the core checks against the rules in the skill:

1. Does the composable take `modifier: Modifier = Modifier`?
2. Is state hoisted, or is there a clear reason to own it here?
3. If it renders a list, does it use a stable `key =`?
4. If it launches work, is that work in a `LaunchedEffect`, `produceState`, or the ViewModel — not in the composition body?
5. If it collects a `Flow`, is it `collectAsStateWithLifecycle()`?
6. Is the parameter order data → `modifier` → other → content slot last?
7. If it animates, is the API declarative first, remembered where needed, lifecycle-aware, and phase-correct?

Any "no" without a reason → fixed before the code comes back. No extra prompt needed — loading the skill is enough.

### Scoped reviews (the token-saver)

The reference files are deliberately loadable in isolation. Scoping a review to one area pulls only that reference into context instead of the full skill.

| You want… | Say |
|---|---|
| state correctness (hoisting, `remember`, saveable, ViewModel) | `compose-agent focus on state` |
| side-effect choice (`LaunchedEffect`, `DisposableEffect`, `produceState`) | `compose-agent focus on effects` |
| recomposition cost + Strong Skipping | `compose-agent focus on performance` |
| modifier hygiene | `compose-agent focus on modifiers` |
| Navigation 3 adoption or Nav2.8 type-safety | `compose-agent focus on navigation` |
| `Flow` collection + lifecycle + coroutine scopes | `compose-agent focus on concurrency` |
| Flow operator selection + `StateFlow`/`SharedFlow` shape | `compose-agent focus on flows` |
| reusable composable API shape | `compose-agent focus on component-api` |
| Compose UI tests, screenshot tests, semantics, previews | `compose-agent focus on testing` |
| focus, keyboard, D-pad, TV/desktop navigation | `compose-agent focus on focus` |
| KMP/CMP source sets, `expect`/`actual`, platform interop | `compose-agent focus on kmp` |
| animation API choice, lifecycle, labels, phase-correct reads | `compose-agent focus on animation` |
| deprecated / soft-deprecated APIs and Android launch resources | `compose-agent focus on api` |
| idiomatic Kotlin / Android style | `compose-agent focus on kotlin` |

### What review mode gives back

- **File-by-file findings.** File + line, rule name, minimal before/after, link to `developer.android.com` or the AndroidX guidelines.
- **Prioritized summary of up to three items**, highest impact first. Act on the chat alone if you are short on time.
- **No nitpicks.** Clean files are not listed.

An example output block is in [`compose-agent/SKILL.md`](./skills/compose-agent/SKILL.md) under "Example Output".

### `compose-agent` vs `jetpack-compose-audit`

| Use `compose-agent`… | Use `jetpack-compose-audit`… |
|---|---|
| while writing or editing a file | for a snapshot of the whole repo |
| for doc-linked fixes on a specific concern | for a 0–100 score across four categories |
| to make the assistant's output *be* correct Compose | to produce a `COMPOSE-AUDIT-REPORT.md` with measured evidence |
| day to day, at file / feature scope | once per release or before a review |

Overlap is fine. Audit on the release candidate, `compose-agent` on every feature branch.

### Layout

```
skills/compose-agent/
  .claude-plugin/plugin.json     Claude Code plugin manifest
  .cursor-plugin/plugin.json     Cursor plugin manifest
  SKILL.md                       short router — loads references on demand
  references/
    api.md                       deprecated + soft-deprecated APIs → modern replacements
    state.md                     hoisting, remember, rememberSaveable, ViewModel boundary
    effects.md                   LaunchedEffect / DisposableEffect / produceState / snapshotFlow
    performance.md               Strong Skipping, lambda modifiers, lazy keys, typed state
    modifiers.md                 order, lambda form, Modifier.Node vs composed { }
    navigation.md                Navigation 3 + Nav2.8 type-safe destinations
    concurrency.md               Flow collection + lifecycle, viewModelScope, dispatchers
    flows.md                     StateFlow / SharedFlow / cold Flow, stateIn, shareIn, flatMap variants, error handling, backpressure
    component-api.md             parameter order, slots, naming, state hoisting shape
    testing.md                   UI tests, semantics assertions, screenshot tests, deterministic fakes, previews
    focus.md                     FocusRequester, keyboard / D-pad input, focus restoration, focus tests
    kmp.md                       KMP/CMP boundaries, expect/actual, interfaces, platform leaf composables
    animation.md                 animation API choice, lifecycle, labels, deferred animated reads
    kotlin.md                    Kotlin conventions + Android Kotlin style the LLM misses
```

---

## Changelog

### 4.0.0 — 2026-06-12

**Added — first-class Compose animation coverage across the suite.**

- **New `compose-agent/references/animation.md`.** Adds API selection (`animate*AsState` vs `updateTransition` vs `Animatable`), remembered animation state, target-driven launches, `spring` / `tween` / `keyframes` guidance, phase-correct animated reads, `AnimatedContent` keys, `AnimatedVisibility`, lazy-list `animateItem()`, and off-screen infinite-transition checks.
- **Review and authoring wiring.** `compose-agent` now has an explicit animation review step, an authoring-mode animation guardrail, a scoped-review entry (`compose-agent focus on animation`), and manifest keywords for animation discovery.
- **Audit reporting clarity.** `jetpack-compose-audit` does not add a fifth Animation category; concrete animation defects continue to affect the existing Performance, Side Effects, or Composable API Quality scores when they violate those rubrics. The report template now surfaces **Animation performance signals** directly inside Performance.
- **Launch materials.** Added release notes and launch-copy drafts in [`docs/release-notes-4.0.0.md`](./docs/release-notes-4.0.0.md) and [`docs/tweet-4.0.0.md`](./docs/tweet-4.0.0.md).
- **Versions.** `compose-agent` → `4.0.0`. `jetpack-compose-audit` → `4.0.0`.

### 3.1.0 — 2026-06-08

**Added — Android Launch UX splash icon audit.**

- **Static Android 12+ splash icon detection.** Normal audits now scan `res/values*/themes.xml` / `styles.xml` for `windowSplashScreenAnimatedIcon`, resolve the referenced resource, and check whether the API 31+ drawable is an `<animated-vector>`.
- **Risk shape.** The audit flags static `<vector>`, `<adaptive-icon>`, bitmap, and layer-list splash icons on Android 12+ when there is no `res/drawable-v31/<name>.xml` animated-vector override.
- **Report shape.** A new non-scored `Android Launch UX` adjacent finding can appear in `COMPOSE-AUDIT-REPORT.md`, `Critical Findings`, and `Prioritized Fixes` when concrete evidence exists.
- **AOSP-grounded framing.** Docs explain the real cause from the platform issue (`520672537`): an `AnimatedVectorDrawable` is `Animatable` and drawn on a SurfaceView at full size, while a static icon goes through `ImmobileIconDrawable`, which pre-renders at `starting_surface_default_icon_size` (108 dp) then upscales to 160/192 dp. Affects XHDPI+ devices since API 31; still open/unfixed at ship time. Includes the official sizing spec (160/192 dp inner circles, 432 dp AVD canvas, 288 dp visible area, 166 ms / ~1000 ms duration caps).
- **Authoring guidance.** `compose-agent` ships a minimal, copy-pasteable workaround verbatim from the reporter: *an `<animated-vector>` with no animators is enough*. Keep the theme `@drawable/<name>` stable and resolve it to an empty `<animated-vector>` wrapping the real vector on API 31+. Documents the one trap that breaks naive attempts — the `drawable-v31` self-reference loop (the wrapper must point at a *differently-named* vector).
- **CI guardrail.** `bin/ci` now fails if either skill drops the Android 12+ static splash icon blur coverage.
- **Versions.** `jetpack-compose-audit` → `3.1.0`. `compose-agent` → `2.1.0`.

### 3.0.0 — 2026-05-29

**Breaking — canonical root `skills/` layout.**

This release removes the compatibility layouts and makes the repository match the modern skill-distribution pattern used by current skill tooling:

```
skills/
  compose-agent/SKILL.md
  jetpack-compose-audit/SKILL.md
```

Why break it: this repo is meant to be an example other Android and Compose skills can copy. Carrying old wrapper folders would make installs harder to explain, make CI weaker, and keep teaching the wrong repository shape. A one-time reinstall is cleaner than preserving layout debt.

Required action for existing installs:

```bash
# npx skills users
npx --yes skills remove compose-agent jetpack-compose-audit -y
npx --yes skills add hamen/compose_skill --skill '*' -y
```

Claude Code users who installed older subdirectories should uninstall the old plugin entry, then reinstall the skills they use:

```
/plugin add hamen/compose_skill --subdir skills/jetpack-compose-audit
/plugin add hamen/compose_skill --subdir skills/compose-agent
```

- **Removed legacy folders.** `compose-agent/`, `jetpack-compose-audit/`, and nested `skills/<plugin>/skills/<name>/` wrappers are gone.
- **Direct skill folders.** `SKILL.md`, `references/`, and `scripts/` now sit directly under `skills/<name>/`.
- **Manifest compatibility retained.** Each direct skill folder still carries `.claude-plugin/plugin.json` and `.cursor-plugin/plugin.json`, so Claude Code and Cursor plugin validation continue to work from the same canonical folder.
- **CI hardened.** `bin/ci` now fails if nested wrapper directories come back, if marketplace paths drift, or if manifest/SKILL versions diverge.
- **No audit behavior change.** Scoring, report shape, references, and Gradle diagnostics are unchanged.
- **Versions.** `jetpack-compose-audit` → `3.0.0`. `compose-agent` → `2.0.0`.

### 2.1.1 — 2026-05-20

**Fixed — Windows plugin loading.**

- **Claude Code plugin layout**: moved both skills into `skills/<name>/SKILL.md` inside their plugin directories and removed the `skills: "./"` manifest field that Windows path normalization rejected.
- **Manual install docs**: symlink commands now target the nested skill directories.
- **Validation**: `claude plugin validate` now passes for both plugin subdirectories.
- **Versions.** `jetpack-compose-audit` → `2.1.1`. `compose-agent` → `1.2.1`.

### 2.1.0 — 2026-05-12

**Added — coverage notes for UI testing, focus navigation, and KMP/CMP.**

- **`compose-agent/references/testing.md`**: plain state-driven Compose UI tests, semantics assertions, callback tests, screenshot tests, deterministic image/platform fakes, and previews.
- **`compose-agent/references/focus.md`**: `FocusRequester`, `focusProperties`, keyboard/D-pad input, key handlers, focus restoration, and focus tests.
- **`compose-agent/references/kmp.md`**: Kotlin Multiplatform / Compose Multiplatform boundaries, semantic common APIs, `expect` / `actual`, fakeable interfaces, platform leaf composables, and native interop.
- **`jetpack-compose-audit`**: maps UI test, screenshot, focus/keyboard, and KMP/CMP surfaces as adjacent coverage notes. The score remains the same four-category 0-100 model, so historical reports stay comparable.
- **Versions.** `jetpack-compose-audit` → `2.1.0`. `compose-agent` → `1.2.0`.

### 2.0.1 — 2026-04-29

**Sharpened — "no IO in composition" rule, with concrete class enumeration.**

- **`compose-agent/references/performance.md`**: expanded the "Expensive Work In Composition" section with explicit IO categories, the Coil/Glide carve-out, the "`remember` is not the fix" warning, O(1) caveats, O(N) string/list work, inline regex triggers, and a "safe in composition" list.
- **`compose-agent/references/effects.md`**: added an anti-pattern pointer back to the performance guidance so effect reviews encounter the composition-time work rule.
- **`jetpack-compose-audit/references/scoring.md`**: tightened Performance and Side Effects deductions around IO and expensive work in composition.
- **`jetpack-compose-audit/references/search-playbook.md`**: added grep patterns for inline regex, O(N) string and collection work, and forbidden IO class names.
- **Versions.** `jetpack-compose-audit` → `2.0.1`. `compose-agent` → `1.1.2`.

### 2.0.0 — 2026-04-28

**Breaking — install URL changed; repo restructured for [agentskills.io](https://agentskills.io/) compliance.**

Both skills in this repo now pass the strict [agentskills.io specification](https://agentskills.io/specification) validator with zero critical findings, audited against [the community validator skill](https://gist.github.com/kingargyle/37e279f27880e7a92253e5f7e7686720) raised in [android/skills#23](https://github.com/android/skills/issues/23). Getting there required moving the audit skill into its own `jetpack-compose-audit/` subdirectory so every skill folder name matches its declared `name`, and removing a cross-skill reference that traversed `..` between the two skills. The cost is a one-time install URL update for existing users.

**Required action for existing installs.**

```
# Old (no longer works)
/plugin add hamen/compose_skill

# New
/plugin add hamen/compose_skill --subdir jetpack-compose-audit
```

Anyone running `/plugin update` against the old install will not pick up changes after `1.5.1`. Reinstall once with `--subdir jetpack-compose-audit` and updates resume normally.

- **Repo restructure.** `SKILL.md`, `references/`, `scripts/`, `.claude-plugin/`, and `.cursor-plugin/` all moved from the repo root into `jetpack-compose-audit/`. Git tracked the moves as renames, so blame and history are preserved. No skill content changed.
- **Cross-skill reference inlined.** `jetpack-compose-audit/references/search-playbook.md` previously linked into `compose-agent/references/flows.md` for the "Flow operators belong outside the composable body" rule — that pointer is replaced with the rule inlined directly. Both skills are now self-contained, with no parent-traversal references between them.
- **Manual symlink instructions updated.** The README symlink target is now `$(pwd)/jetpack-compose-audit` instead of `$(pwd)`.
- **`compose-agent` is unchanged.** Folder, install URL (`/plugin add hamen/compose_skill --subdir compose-agent`), and version (`1.1.1`) are all the same. No action needed for compose-agent users.

**Why 2.0.0 and not 1.6.0.** The install URL change is breaking — users on the old URL silently stop receiving updates. Semver says that's a major bump, regardless of whether the rubric or report content moved. Actual audit behavior, scoring, categories, and report format are identical to `1.5.1`.

### 1.5.1 — 2026-04-27

**Fixed — Flow guidance consistency.**

This patch keeps the `1.5` Flow reference intact, but tightens the wording so agents do not learn two subtly different rules from adjacent sections.

- **Must-deliver outcomes stay durable.** The `StateFlow` / `SharedFlow` decision table no longer lists in-app purchase results as generic one-shot events. Payment, deletion, save, and similar outcomes that must not be lost are consistently described as durable UI state with an acknowledgement callback.
- **Effect collection keys clarified.** Manual event collection now says to key `LaunchedEffect` on the changing flow identity, and to use `LaunchedEffect(Unit)` only when the flow object is intentionally stable for the call-site lifetime.
- **Search heuristic softened.** `collectLatest` is no longer grouped with Flow-shaping operators that automatically belong in presenters. Terminal collection directly in a composable body is still a smell; collection inside a correctly keyed `LaunchedEffect(...)` for ephemeral events is explicitly acceptable.
- **Audit command safer on Linux.** The Flow-misuse search pipeline now uses `xargs -r` so an empty composable-file list does not accidentally run the second search across the whole repo.

### 1.5.0 — 2026-04-27

**Added — Flow operator reference.** `compose-agent/references/flows.md` is the missing counterpart to `concurrency.md`: lifecycle, scopes, and dispatchers stay in `concurrency.md`; operator choice and exposed Flow shape now live in `flows.md`.

This release closes the "Flow effectively" gap from [android/skills#27](https://github.com/android/skills/issues/27). Compose, Coroutines, and Navigation 3 were already covered by `compose-agent`; Flow operator selection was the weak spot.

What changed:

- **Flow shape decisions.** Clear guidance for `StateFlow` vs `SharedFlow` vs cold `Flow`, including when UI outcomes should be durable state instead of ephemeral events.
- **`stateIn` / `shareIn` correctness.** ViewModel state uses `stateIn(scope, started, initialValue)` with `WhileSubscribed(5_000)` as the default recommendation. `shareIn` is documented with its real API shape: `scope`, `started`, and `replay`; buffering belongs upstream via `buffer(...)` / `conflate()`, not as nonexistent `shareIn` parameters.
- **Operator selection.** Covers `flatMapLatest` / `flatMapMerge` / `flatMapConcat`, `combine` / `merge` / `zip`, `catch` / `retry` / `retryWhen` / `onCompletion`, and backpressure via `buffer`, `conflate`, and `collectLatest`.
- **Nonexistent APIs called out.** The reference explicitly says `kotlinx.coroutines.flow` does not ship `throttleLatest` / `throttleFirst`; agents should not invent imports for them.
- **Mutable vs read-only exposure.** Reinforces `asStateFlow()`, `asSharedFlow()`, and `receiveAsFlow()` so agents do not expose mutable primitives or call `tryEmit` on read-only `SharedFlow`.
- **Audit wording tightened.** Scoring categories, weights, and report format are unchanged, but the state-management rubric now follows Android's UI-events guidance: model must-deliver UI outcomes as state with acknowledgement; reserve `Channel` / `SharedFlow` for ephemeral signals with documented delivery semantics.

**Wiring.** Added as Review Process step 8 in `compose-agent/SKILL.md`, between concurrency and composable API review. The README, layout docs, scoped-review table, and both Claude/Cursor plugin manifests were updated for the `1.5.0` / `1.1.0` release.

### 1.4.1 — 2026-04-24

**Fixed — `compose-agent` documentation correctness.**

This is a same-day follow-up to `1.4`. The new sibling skill shipped useful guidance, but a few reference notes were too absolute or technically wrong. `1.4.1` tightens those sections against the current official Android and Kotlin docs so the skill stops teaching bad fixes in authoring mode.

- **Performance reference corrected.** `remember` semantics now match the docs: the calculation runs during the first composition, then again only when the call leaves composition or its keys change. The bogus `remember { stringResource(...) }` suggestion is gone, the nonexistent "lambda-form padding" recommendation is gone, and the sample state-reader lambda now compiles.
- **Concurrency reference corrected.** `MutableStateFlow.value` is no longer described as main-thread-confined. The guidance now reflects the official thread-safe semantics and frames `.value`, `emit(...)`, and `update { ... }` by API shape rather than folklore.
- **State reference softened.** Snapshot lists/maps are still recommended for in-place UI mutations, but immutable list replacement via `StateFlow<List<T>>` / `MutableState<List<T>>` is now called out as equally valid architecture.
- **Navigation 3 reference corrected.** Saveable state and NavEntry-scoped `ViewModel`s are now described in terms of `NavEntryDecorator` setup, matching current `NavDisplay` defaults and the `rememberViewModelStoreNavEntryDecorator()` add-on.
- **Custom modifier example fixed.** The zero-parameter `ModifierNodeElement` sample now uses the singleton/no-parameter pattern from the docs, and the stateful `Checkbox` convenience example now compiles.
- **Version bumps.** The audit skill manifests now read `1.4.1`; the sibling `compose-agent` manifests now read `1.0.1`.

### 1.4 — 2026-04-24

**Added — `compose-agent` sibling skill.** An agent skill that helps AI coding assistants write modern Jetpack Compose.

Where `jetpack-compose-audit` scores your whole repo end-to-end, `compose-agent` works at file and feature scope while you are reading, writing, or modifying Compose. It targets the mistakes LLMs actually make. Built as a response to [android/skills#27](https://github.com/android/skills/issues/27) — the Android equivalent of [SwiftUI Pro](https://github.com/twostraws/swiftui-agent-skill).

- **Short `SKILL.md` that routes to nine focused reference files** — `api`, `state`, `effects`, `performance`, `modifiers`, `navigation`, `concurrency`, `component-api`, `kotlin`. Only the reference relevant to the current task is pulled into context.
- **Two modes, no config.** Review mode: "check this file" returns a file-by-file report with before/after snippets and official doc links. Authoring mode: six silent guardrails run on every new composable before the code comes back (`modifier` param, state hoisting, lazy keys, effect placement, lifecycle-aware collection, parameter order).
- **Scoped reviews save tokens.** `compose-agent focus on state` loads only `state.md` — roughly a tenth of a full review. Same for `effects`, `performance`, `modifiers`, `navigation`, `concurrency`, `component-api`, `api`, `kotlin`.
- **Install side by side** with the audit skill: `/plugin add hamen/compose_skill --subdir compose-agent`. Cursor and manual-symlink paths are documented in the new "Sibling skill" section of this README.

**Proven on a real app.** Dogfooded against `kindle-gratis-compose` before shipping. In one review of three files it caught:

- A `Random.nextInt()` called in an `@Composable` body — every recomposition picked a new placeholder URL and re-requested the image.
- `Color.Black` text laid over a blended `colorScheme.primary` + `colorScheme.surface` background — unreadable in dark mode. Light-mode tests do not catch this.
- A `DisposableEffect` + `LifecycleEventObserver` for an `ON_START` refresh — should be `LifecycleStartEffect` from lifecycle-runtime-compose 2.8+. Half the code, same behavior.
- Five composables with `modifier` either missing or positioned before required data (AndroidX component guideline violation).
- Hardcoded English strings in `contentDescription` / `onClickLabel` where the rest of the file already used `stringResource`.

Every finding came with a file:line, a before/after, and a link to the official AndroidX or `developer.android.com` page behind the rule. Top-three prioritized summary at the end.

**Added — dark-mode contrast heuristic** (driven by the dogfood run above). Hard-coded foreground colors (`Color.Black`, `Color.White`, raw ARGB literals) over theme-derived backgrounds (`colorScheme.*` or blends thereof) are now flagged as a dark-mode regression risk. Documented in `compose-agent/references/component-api.md` with a "Dark Mode Correctness" section, grep triggers, and a fix pattern that reads color from the matching `on*` role.

**Added — `LifecycleStartEffect` / `LifecycleResumeEffect` guidance.** The lifecycle-runtime-compose 2.8+ APIs replace the verbose `DisposableEffect` + `LifecycleEventObserver` pattern. Covered in `compose-agent/references/effects.md` as a first-class side-effect option and in `references/api.md` as a soft-deprecation flag. The old idiom is pervasive in LLM-generated Compose code — exactly the kind of post-training-cutoff API the skill exists to teach.

**No change to the audit skill rubric.** `jetpack-compose-audit` behavior, categories, scoring, and report format are identical to `1.3`. This release is purely additive. Run the audit once per release; run `compose-agent` every day.

### 1.3 — 2026-04-22

**Added — Cursor plugin support.**

Same-day follow-up to `1.2`. Cursor's plugin system is structurally identical to Claude Code's (hidden manifest directory, single-skill plugins auto-discover a root-level `SKILL.md`), so shipping a sidecar manifest was essentially free — and it unblocks org admins who want to import the skill into their team marketplace via Cursor's GitHub-import flow.

- **New file**: `.cursor-plugin/plugin.json` at the repo root. Same metadata as the Claude Code manifest (name, version, description, author, repository, license, keywords). The `skills` field is intentionally omitted so Cursor treats the repo as a single-skill plugin and auto-discovers `SKILL.md` at the root — matching the documented default behaviour.
- **Install docs**: the README Install section is restructured around harnesses (Claude Code / Cursor / manual symlink) rather than recommended-vs-fallback. Each path is clearly labelled with what it does and does not unlock today.
- **Not submitted to Cursor Marketplace (yet)**. The manifest is a prerequisite for submission and for team-marketplace imports, but no public Marketplace listing was created in this release. Individual Cursor users should continue with the symlink install until that changes.
- **Version alignment**: `.claude-plugin/plugin.json` bumped to `1.3.0` to match. Both plugin manifests and `SKILL.md` now read the same version string.

### 1.2 — 2026-04-22

**Added — Claude Code plugin support.**

The skill now ships a `.claude-plugin/plugin.json` manifest, so Claude Code users can install it directly from the Git URL via `/plugin add hamen/compose_skill` instead of cloning and symlinking by hand. Nothing else about the skill changed — same rubric, same categories, same report template.

- **New file**: `.claude-plugin/plugin.json` at the repo root. Declares the skill name, version, repository, license, and keywords, and points Claude Code at the existing `SKILL.md` via `"skills": "./"`. Purely additive — no existing files were restructured.
- **README install section rewritten**: Claude Code plugin install is now the primary path. The previous symlink flow is kept verbatim as a **Manual install / Cursor** fallback for Cursor users and for older Claude Code versions that don't yet support the plugin system.
- **Why now**: tracks [android/skills#7](https://github.com/android/skills/issues/7) (47 👍 at time of writing) — install friction was the single biggest piece of community feedback on the Agent Skills install story. Same bottleneck applied here; same fix.

### 1.1.1 — 2026-04-21

**Refined — Strong Skipping-aware scoring, detection, and reporting.**

This is a corrective follow-up to `1.1`. After feedback around Strong Skipping Mode, the skill now evaluates modern Compose repos the way the compiler actually behaves on Kotlin `2.0.20+` / Compose Compiler `1.5.4+`.

- **Performance rubric**: split the measured-ceiling logic into explicit **SSM-off** and **SSM-on** paths. On older compiler tracks, ceilings still depend on `skippable%` and unstable shared params. Under Strong Skipping, the audit no longer blindly caps scores because a repo uses raw `List` params or misses `@Stable` on its own.
- **What now matters under SSM**: the audit now looks for the issues that still materially defeat skipping in practice: per-recomposition param churn (`listOf(...)`, `mapOf(...)`, fresh UI models, object / lambda literals in composable bodies), expensive or broken `equals()` on unstable params, and unjustified `@NonSkippableComposable` / `@DontMemoize` opt-outs on hot paths.
- **More honest compiler interpretation**: the docs now explicitly distinguish module-wide `skippable%` from **named-only `skippable%`**, because zero-argument lambdas can artificially drag down the raw module number. Reports are instructed to say which metric actually bound the ceiling.
- **Collection guidance corrected**: `ImmutableList` / `PersistentList` are no longer framed as a mandatory cargo-cult fix under Strong Skipping. They still earn credit, but for the right reasons: structural sharing, predictable equality, and lower churn when collections are reused deliberately.
- **Search playbook**: the Strong Skipping section now tells the agent exactly how to detect explicit opt-ins / opt-outs, when **not** to deduct for raw `List` params or missing `@Stable`, and what to inspect instead when SSM is active.
- **Diagnostics**: Strong Skipping is now resolved **per module**, not assumed once for the whole repo. The diagnostics docs cover default-on, explicit opt-out, older-version opt-in, and mixed-module repos where different modules may require different ceiling tables.
- **Report template**: added a mandatory **Performance ceiling check** block so every report records Strong Skipping status, the table applied, module-wide vs named-only `skippable%`, unstable shared-type count, the actual binding cap, and the final applied score.
- **Chat summary guidance**: the short final summary now mirrors the same SSM-aware framing as the report. Under SSM, expected impact is described in terms of removing churn, fixing `equals()`, or clearing the binding cap rather than promising a magic `skippable%` jump.
- **README examples and docs**: refreshed the public-facing example output to show a realistic SSM-era case: Strong Skipping on, high named-only skippability, and a score capped by recreated params rather than by the raw module-wide percentage.

**Net effect:** fewer false positives on modern Compose codebases, fewer cargo-cult recommendations, and more defensible audit reports when the repo already runs with Strong Skipping enabled by default.

### 1.1 — 2026-04-21

**Added — animation auditing.**

- **Performance**: new rules for animation phase correctness. Flags animated `.value` reads piped into state-reading modifiers (`Modifier.offset(x.dp)`, `Modifier.alpha(a)`) when lambda-form modifiers (`Modifier.graphicsLayer { ... }`, `Modifier.offset { ... }`) would defer to layout/draw. Flags `Animatable(...)` created in a composable body without `remember { ... }` or hoisting. Flags `rememberInfiniteTransition()` hosted in composables that stay composed offscreen. Adds API-choice guidance for `Crossfade` vs `AnimatedContent`: standard fades remain fine with `Crossfade`, while custom enter/exit or size-aware swaps belong in `AnimatedContent`.
- **Side effects**: new rules for animation driving. Flags `Animatable` animations launched from the composition body. Flags `rememberCoroutineScope().launch { animatable.animateTo(...) }` when the animation is target-driven and `LaunchedEffect(target)` is the clearer fit.
- **Composable API quality**: new rules for reusable animated components. Encourages exposing `animationSpec: AnimationSpec<T>` on shared animated APIs when callers may reasonably need timing control. Treats missing `label` parameters on tooling-visible shared animations as a light tooling-quality smell rather than a hard failure.
- **Search playbook**: dedicated "Animation Phase-Smell Heuristic" section and new grep patterns for animation APIs.
- **Canonical sources**: added the Compose Animation docs (`animation/introduction`, `animation/value-based`, `animation/customize`) to the URL ledger every deduction must cite.

### 1.0 — initial release

- Four scoring categories: Performance (35%), State management (25%), Side effects (20%), Composable API quality (20%).
- Automatic Compose Compiler reports via a bundled Gradle init script (`scripts/compose-reports.init.gradle`).
- Measured `skippable%` ceilings on the Performance score.
- Every deduction cited against `developer.android.com` or the AndroidX component guidelines.
- Mirrored chat summary and written `COMPOSE-AUDIT-REPORT.md` with prioritized fixes.

---

## License

MIT.
