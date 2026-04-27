# Jetpack Compose Audit Skill

**Version 1.5.1 · released 2026-04-27** — Patch release tightening Flow, event-stream, and effect-keying guidance for `compose-agent`.

> Find out where your Compose app is burning frames, by how much, and what to change to win them back — measured against real compiler data, not vibes.

A strict, evidence-based audit for Android Jetpack Compose repositories. Point it at a repo, let it run the build once, and get back a 0-100 score, a 0-10 score per category, an actionable top-three fix list, and a full Markdown report with every deduction cited against an official `developer.android.com` page.

Built for Claude Code, Cursor, and any agent that loads the Anthropic skill format.

---

## Changelog

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

## What you get

Run the skill on a Compose repo and you walk away with:

- **`COMPOSE-AUDIT-REPORT.md`** written at the target root — per-category scoring, evidence file paths, line numbers, and prioritized fixes.
- **A chat summary** that mirrors the report's top three fixes — same file paths, same doc links, same predicted impact. Act on the chat alone if you're short on time.
- **Measured stability numbers** from the Compose Compiler — module-wide `skippable%`, named-only `skippable%`, the unstable-class list, and the per-module Strong Skipping state inferred from compiler version plus explicit flags.
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

The report lists every occurrence with file path and line number, not just the category.

---

## What makes it different

**Measured, not inferred.** The skill ships `scripts/compose-reports.init.gradle` and injects it into your Gradle build via `--init-script` — no edits to your `build.gradle`. Every run parses real `*-classes.txt` / `*-composables.txt` / `*-module.json` output.

**Mandatory ceilings.** A Performance score cannot exceed the cap set by the matching ceiling table. On older compiler tracks the cap is driven by `skippable%` plus unstable-param count; under Strong Skipping it is driven by named-only `skippable%`, instance-recreation churn, and `equals()` quality on unstable params. The ceiling math appears in the report so the score is auditable.

**Every deduction cites an official source.** Each finding carries a `References:` line pointing at `developer.android.com` or the AndroidX component API guidelines. Audits that can't be defended with a URL don't ship.

**Actionable chat summary.** The chat output mirrors the report's `Prioritized Fixes` — same file paths, same doc links, same predicted impact ("stops rebuilding `FeedItemUiModel`, removes the Strong-Skipping cap from 8 → no cap").

---

## Install

### Claude Code

Install directly from the Git repository — no cloning, no symlinking:

```
/plugin add hamen/compose_skill
```

Claude Code reads `.claude-plugin/plugin.json` and registers `SKILL.md` automatically. Updates arrive via the normal plugin update flow.

### Cursor

The repo ships a `.cursor-plugin/plugin.json` manifest, so Cursor sees it as a valid single-skill plugin. There are two install paths today:

- **Team Marketplace (orgs)**: a workspace admin can import this GitHub repository into a team marketplace. The manifest is what makes that import succeed.
- **Manual symlink (individuals)**: use the manual install block below. Individual Cursor users do not have a `/plugin add`-style command for arbitrary Git URLs yet; symlink install remains the practical path until this skill is published to the public Cursor Marketplace.

### Manual install (all harnesses)

Use this on Cursor, on older Claude Code versions without `/plugin add`, or whenever you want `git pull` in this directory to update the skill in place:

```bash
# Claude Code
mkdir -p ~/.claude/skills
ln -s "$(pwd)" ~/.claude/skills/jetpack-compose-audit

# Cursor
mkdir -p ~/.cursor/skills
ln -s "$(pwd)" ~/.cursor/skills/jetpack-compose-audit
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

**In scope (1.x).** Jetpack Compose on Android, Kotlin 2.0.20+ / Compose Compiler 1.5.4+ (Strong Skipping default).

**Out of scope (1.x)** — the skill will call these out as a note rather than silently produce thin coverage:

- Material 3 compliance, theming, color/typography — defer to the `material-3` skill.
- Accessibility scoring (semantics, touch targets) — flagged as notes, not scored.
- UI test coverage and Compose test-rule patterns.
- Compose Multiplatform (`expect`/`actual`, target-specific code paths).
- Wear OS / TV / Auto / Glance surfaces.
- Build performance (incremental compilation, KSP/KAPT choice).

---

## Layout

```
SKILL.md                         main audit skill (process, principles, output)
scripts/
  compose-reports.init.gradle    Gradle init script injected via --init-script
references/
  scoring.md                     rubric with measured ceilings and inline citations
  search-playbook.md             grep patterns, regex, read-the-file heuristics
  canonical-sources.md           every URL the rubric cites
  report-template.md             required structure for COMPOSE-AUDIT-REPORT.md
  diagnostics.md                 manual-mode fallback snippets
```

A sibling skill, `compose-agent/`, ships in the same repo — see [§ Sibling skill](#sibling-skill--compose-agent) for its own layout and usage.

---

## Philosophy

- **Strict but evidence-based.** Every deduction has a file:line and an official-doc URL.
- **Measured beats inferred.** Compiler reports are generated automatically; source-inferred stability is a fallback, not the default.
- **Written for action.** The report's `Prioritized Fixes` section and the chat summary mirror each other, so the developer can act on the chat alone.
- **Narrow scope on purpose.** The skill does not score design, accessibility, or build performance in v1. It says so rather than pretending otherwise.

---

## Sibling skill — `compose-agent`

This repo ships a second skill alongside the audit: [`compose-agent/`](./compose-agent/). Where the audit **reviews an existing repo** end-to-end and produces a score, `compose-agent` works at **file and feature scope** while you are reading, writing, or modifying Compose.

- **Responds to:** "is this right?", "rewrite this the modern way", "check this file for deprecated API", "find state hoisting mistakes in this feature".
- **Built for:** [android/skills#27](https://github.com/android/skills/issues/27) — the Android equivalent of [`swiftui-agent-skill`](https://github.com/twostraws/swiftui-agent-skill). The philosophy is the same: target the mistakes LLMs actually make in Compose, not repeat basics the model already knows.
- **Shape:** short `SKILL.md` that routes to ten per-topic reference markdowns. You only pay the token cost for the areas your current task touches.

### Install

Same flow as the audit skill, pointing at the subdirectory.

**Claude Code:**

```
/plugin add hamen/compose_skill --subdir compose-agent
```

**Cursor:** import the repo as a plugin and pick `compose-agent` in the subdirectory selector.

**Manual:** symlink `compose-agent/` into your skills directory (`~/.claude/skills/compose-agent`, `~/.cursor/skills/compose-agent`, etc.).

Both skills can live side by side — they do not share state and do not interfere.

### Use it — the two modes

`compose-agent` runs in **review mode** or **authoring mode**. You do not choose; the skill picks based on your request.

**Review mode** — you hand it code that already exists. It produces a file-by-file report with before/after snippets and links to the official doc page behind each rule. Triggered by prompts like:

```
Use compose-agent to review feature/profile/.
Check ProfileScreen.kt with compose-agent.
compose-agent: find deprecated API in this module.
```

**Authoring mode** — the skill is loaded and you ask the assistant to *write* Compose. Before returning code, the assistant silently runs six checks against the rules in the skill:

1. Does the composable take `modifier: Modifier = Modifier`?
2. Is state hoisted, or is there a clear reason to own it here?
3. If it renders a list, does it use a stable `key =`?
4. If it launches work, is that work in a `LaunchedEffect`, `produceState`, or the ViewModel — not in the composition body?
5. If it collects a `Flow`, is it `collectAsStateWithLifecycle()`?
6. Is the parameter order data → `modifier` → other → content slot last?

Any "no" without a reason → fixed before the code comes back. No extra prompt needed — loading the skill is enough.

### Scoped reviews (the token-saver)

The ten reference files are deliberately loadable in isolation. Scoping a review to one area pulls only that reference into context — a focused review costs roughly a tenth of a full one.

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
| deprecated / soft-deprecated APIs | `compose-agent focus on api` |
| idiomatic Kotlin / Android style | `compose-agent focus on kotlin` |

### What review mode gives back

- **File-by-file findings.** File + line, rule name, minimal before/after, link to `developer.android.com` or the AndroidX guidelines.
- **Prioritized summary of up to three items**, highest impact first. Act on the chat alone if you are short on time.
- **No nitpicks.** Clean files are not listed.

An example output block is in [`compose-agent/SKILL.md`](./compose-agent/SKILL.md) under "Example Output".

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
compose-agent/
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
    kotlin.md                    Kotlin conventions + Android Kotlin style the LLM misses
```

---

## License

MIT.
