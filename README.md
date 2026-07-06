<h1 align="center">Compose Skill Suite</h1>

<p align="center">
  <strong>Jetpack Compose audit + coding-agent skills for Claude Code, Codex, Cursor, and Anthropic-style skill loaders.</strong>
</p>

<p align="center">
  <a href="./bin/ci"><img alt="bin/ci passing" src="https://img.shields.io/badge/bin%2Fci-passing-2ea043"></a>
  <a href="https://github.com/hamen/compose_skill/releases"><img alt="Release" src="https://img.shields.io/github/v/release/hamen/compose_skill?color=2f80ed&label=release"></a>
  <a href="LICENSE"><img alt="License" src="https://img.shields.io/github/license/hamen/compose_skill?color=0a7f60"></a>
  <img alt="Skills" src="https://img.shields.io/badge/skills-2-7c3aed">
  <img alt="Compose animation ready" src="https://img.shields.io/badge/Compose-animation%20ready-f97316">
  <img alt="Claude Code plugin" src="https://img.shields.io/badge/Claude%20Code-plugin-111827">
</p>

**`jetpack-compose-audit` 4.3.0 · 2026-07-06** — Cross-phase back-write detection (layout callbacks writing state read in composition; snapshot collections mutated in a composable body) plus a **False Leads** scoring guard so the auditor stops crediting no-op "recomposition fixes." Axis 3 of the recomposition problem, adapted from [`chrisbanes/skills`](https://github.com/chrisbanes/skills) (Apache-2.0). `compose-agent` stays at `4.2.1`.

**Version 4.2.0 · 2026-06-17** — Paging 3 in Compose: new `paging.md` reference (LLM guardrails, not API tour), audit hooks under existing Performance/State categories, planning doc at [`docs/paging-skill-plan.md`](./docs/paging-skill-plan.md). Validated through multi-agent cross-review. Both skills ship as `4.2.0`.

> Find out where your Compose app is burning frames, by how much, and what to change to win them back — measured against real compiler data, not vibes.

A strict, evidence-based audit for Android Jetpack Compose repositories. Point it at a repo, let it run the build once, and get back a 0-100 score, a 0-10 score per category, an actionable top-three fix list, and a full Markdown report with every deduction cited against an official `developer.android.com` page.

Built for Claude Code, Codex, Cursor, and any agent that loads the Anthropic skill format.

Authored and cross-reviewed with every frontier model — Claude Opus 4.8, GPT-5.5, and Gemini 3.5 Flash — each used to pressure-test the rubric, prompts, and references so the audit holds up regardless of which model ends up running it.

---

## What's new

### 4.3.0 — 2026-07-06

**`jetpack-compose-audit` — cross-phase back-writes + false-lead guard.**

- **Cross-phase back-write detector.** Beyond same-body backwards writes, the audit now flags a later phase writing snapshot state an earlier phase read: `onSizeChanged` / `onGloballyPositioned` / `onPlaced` (layout) writing state a sibling reads in composition, or `SnapshotStateMap` / `SnapshotStateList` mutated inside a `@Composable` body. New **Cross-Phase Back-Write Heuristic** in [`search-playbook.md`](./skills/jetpack-compose-audit/references/search-playbook.md), matching Performance deduction in [`scoring.md`](./skills/jetpack-compose-audit/references/scoring.md), triage step 7 in [`diagnostics.md`](./skills/jetpack-compose-audit/references/diagnostics.md). Findings cite writer and reader `file:line`.
- **False Leads guard.** New **Do Not Credit — False Leads** table in `scoring.md`: plausible recomposition "fixes" that change nothing. The auditor no longer rewards them and will not suggest them in `Prioritized Fixes`.
- **Attribution.** Axis 3 and the false-lead cases adapted (reworded, re-cited against `developer.android.com`) from [`chrisbanes/skills`](https://github.com/chrisbanes/skills) `compose-recomposition-performance` (Apache-2.0). Axes 1–2 were already covered by this suite.
- **Versions.** `jetpack-compose-audit` → `4.3.0`. `compose-agent` unchanged at `4.2.1`.

### 4.2.1 — 2026-06-29

**`compose-agent` — Navigation 3 decision table.**

- **Workflow entry point.** A "when to use" decision table at the top of [`navigation.md`](./skills/compose-agent/references/navigation.md): Nav3 vs Nav2 type-safe vs plain state, argument passing, ViewModel-triggered navigation (the route mutates the back stack — not a screen ViewModel), results via Nav3's `ResultEventBus`, per-entry `ViewModel` scope, `BackHandler`, deep links, and `ListDetailSceneStrategy` vs `SinglePaneSceneStrategy`. Mirrors the `paging.md` pattern. From [android/skills#50](https://github.com/android/skills/issues/50). API shapes verified against [`android/nav3-recipes`](https://github.com/android/nav3-recipes).
- **Versions.** `compose-agent` → `4.2.1`. `jetpack-compose-audit` unchanged at `4.2.0`.

### 4.2.0 — 2026-06-17

**Paging 3 in Compose — guardrails for paged lazy lists.**

- **New reference.** [`skills/compose-agent/references/paging.md`](./skills/compose-agent/references/paging.md) — decision table, golden path, stable keys, `LoadState`, hard nos. Targets LLM mistakes, not `PagingSource` implementation.
- **Review wiring.** New step #14 in `compose-agent`; scoped entry `compose-agent focus on paging`.
- **Audit hooks.** Report signals **Paging list correctness** (Performance) and **Paging load-state handling** (State) — no fifth score category.
- **Plan.** [`docs/paging-skill-plan.md`](./docs/paging-skill-plan.md) includes the multi-agent validation checklist.
- **Versions.** `compose-agent` → `4.2.0`. `jetpack-compose-audit` → `4.2.0`.

For release detail, see [`docs/release-notes-4.2.0.md`](./docs/release-notes-4.2.0.md). Full history: [CHANGELOG.md](./CHANGELOG.md).

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

Add this repository as a plugin marketplace, then install either plugin from it.

From inside an interactive Claude Code session:

```text
/plugin marketplace add hamen/compose_skill
/plugin install compose-agent@compose_skill
/plugin install jetpack-compose-audit@compose_skill
```

Or from your shell:

```bash
claude plugin marketplace add hamen/compose_skill
claude plugin install compose-agent@compose_skill
claude plugin install jetpack-compose-audit@compose_skill
```

The root `.claude-plugin/marketplace.json` points each plugin at its `skills/<name>` subdir, where Claude Code reads `.claude-plugin/plugin.json` and registers `SKILL.md`.

### Breaking migration

The supported Claude Code flow is the `marketplace add` + `install` commands above. Note the `claude plugin` **CLI** has no `add` subcommand, so the shell form `claude plugin add … --subdir` does not work — use the marketplace commands instead.

If you previously installed via an older `--subdir` path or the nested `skills/<plugin>/skills/<name>` manual symlink, **uninstall the old plugin first** (otherwise `/plugin update` keeps pointing at the stale path), then reinstall with the marketplace commands above.

First list what you have — a legacy install may appear under a different id than `@compose_skill`:

```text
/plugin list
```

Then uninstall whatever old entry it shows (use the exact id from the list), e.g.:

```text
/plugin uninstall compose-agent@compose_skill
/plugin uninstall jetpack-compose-audit@compose_skill
```

The same commands work from your shell as `claude plugin list` and `claude plugin uninstall <id>`.

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

```text
/plugin marketplace add hamen/compose_skill
/plugin install compose-agent@compose_skill
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
| Paging 3 in Compose (`LazyPagingItems`, keys, `LoadState`) | `compose-agent focus on paging` |
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
    paging.md                    Paging 3 in Compose: keys, LoadState, refresh/retry guardrails
    kotlin.md                    Kotlin conventions + Android Kotlin style the LLM misses
```

---

## License

MIT.
