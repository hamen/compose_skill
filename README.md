# Jetpack Compose Audit Skill

**Version 1.1 · released 2026-04-21**

> Find out where your Compose app is burning frames, by how much, and what to change to win them back — measured against real compiler data, not vibes.

A strict, evidence-based audit for Android Jetpack Compose repositories. Point it at a repo, let it run the build once, and get back a 0-100 score, a 0-10 score per category, an actionable top-three fix list, and a full Markdown report with every deduction cited against an official `developer.android.com` page.

Built for Claude Code, Cursor, and any agent that loads the Anthropic skill format.

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

Symlink the repo into your skills directory so `git pull` updates it everywhere:

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

## Changelog

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

## Layout

```
SKILL.md                         main skill manifest (process, principles, output)
scripts/
  compose-reports.init.gradle    Gradle init script injected via --init-script
references/
  scoring.md                     rubric with measured ceilings and inline citations
  search-playbook.md             grep patterns, regex, read-the-file heuristics
  canonical-sources.md           every URL the rubric cites
  report-template.md             required structure for COMPOSE-AUDIT-REPORT.md
  diagnostics.md                 manual-mode fallback snippets
```

---

## Philosophy

- **Strict but evidence-based.** Every deduction has a file:line and an official-doc URL.
- **Measured beats inferred.** Compiler reports are generated automatically; source-inferred stability is a fallback, not the default.
- **Written for action.** The report's `Prioritized Fixes` section and the chat summary mirror each other, so the developer can act on the chat alone.
- **Narrow scope on purpose.** The skill does not score design, accessibility, or build performance in v1. It says so rather than pretending otherwise.

---

## License

MIT.
