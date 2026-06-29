# Changelog

Full release history for the Compose Skill Suite. The newest release is summarised under **What's new** in the [README](./README.md).

### 4.2.1 — 2026-06-29

**`compose-agent` only — Navigation 3 decision table.**

- **Workflow entry point.** Added a "when to use" decision table to `skills/compose-agent/references/navigation.md`: Nav3 vs Nav2 type-safe vs plain state, argument passing, ViewModel-driven navigation, per-entry `ViewModel` scope, `BackHandler`, deep links, and `ListDetailSceneStrategy` vs `SinglePaneSceneStrategy`. Mirrors the `paging.md` decision-table pattern. Addresses [android/skills#50](https://github.com/android/skills/issues/50) (Nav3 was index-only without a workflow entry point).
- **Versions.** `compose-agent` → `4.2.1`. `jetpack-compose-audit` unchanged at `4.2.0`.

### 4.2.0 — 2026-06-17 (release candidate — convergence testing before GA tag)

**Paging 3 in Compose — guardrails for paged lazy lists.**

- **New reference.** Added `skills/compose-agent/references/paging.md` — when to page vs plain lists, `collectAsLazyPagingItems`, stable `itemKey`, `LoadState` UI, user-driven `refresh()` / `retry()`, anti-pattern checklist. No `PagingSource` / RemoteMediator cookbook.
- **Review wiring.** `compose-agent` review step #14, core instruction, authoring guardrail, manifest keyword `paging`.
- **Audit hooks.** `jetpack-compose-audit` adds paging search heuristic plus **Paging list correctness** (Performance) and **Paging load-state handling** (State) report signals — no fifth score category.
- **Planning.** [`docs/paging-skill-plan.md`](./docs/paging-skill-plan.md) records scope and multi-agent validation plan.
- **Versions.** `compose-agent` → `4.2.0`. `jetpack-compose-audit` → `4.2.0`.

### 4.1.2 — 2026-06-14

**Repo hygiene — evals as data, changelog split out, launch cruft removed.**

- **Evals are machine-readable now.** The 4.1.0 acceptance scenarios moved from prose in `docs/evals-4.1.0.md` to `skills/jetpack-compose-audit/evals/evals.json` — `skill_name` plus ten `{prompt, expected_output, expectations}` cases, in an `evals/` directory inside the skill (the canonical eval layout).
- **Changelog split out.** This `CHANGELOG.md` now holds the full release history; the README keeps only the latest release under **What's new**.
- **Dropped launch cruft.** Removed the launch-tweet drafts (`docs/tweet-4.0.0.md`, `docs/tweet-4.1.0.md`). Release notes stay in `docs/`.
- **No behavior change.** Skill guidance, audit scoring, and search leads are identical to `4.1.1`.
- **Versions.** `compose-agent` → `4.1.2`. `jetpack-compose-audit` → `4.1.2`.

### 4.1.1 — 2026-06-14

**Patch — search-lead and docs fixes.**

- **Fixed.** The non-`Unit` composable search lead in `jetpack-compose-audit/references/search-playbook.md` no longer stops at the first `)`, so composables with lambda parameters are matched. Documented as eyeball-verify since the broadened span can over-match.
- **Fixed.** Hard line break restored on the Kotlin coroutines best-practices entry in `references/canonical-sources.md` so the URL renders on its own line.
- **Versions.** `compose-agent` → `4.1.1`. `jetpack-compose-audit` → `4.1.1`.

### 4.1.0 — 2026-06-14

**Eval-driven — API hygiene and effect-correctness checks.**

- **Eval set.** Added [`docs/evals-4.1.0.md`](./docs/evals-4.1.0.md) with the scenarios that drove this release. These are the release's acceptance criteria, not just examples.
- **Composable API contracts.** `compose-agent/references/component-api.md` now covers callback-vs-slot placement, event callback naming, slot lifecycle when content moves, UI-emitting composables that return values, multiple sibling roots without parent scopes, pure helpers marked `@Composable`, and `@ReadOnlyComposable` misuse.
- **Effects and cancellation.** `compose-agent/references/effects.md` and `concurrency.md` now catch eager `rememberUpdatedState` reads inside `remember`, leaf UI launching durable business work, and broad failure handlers / `runCatching` that swallow `CancellationException`.
- **Audit scoring/search.** `jetpack-compose-audit` now has search leads and scoring deductions for the same issues, plus a canonical source entry for Kotlin coroutine cancellation best practices.
- **Scratch hygiene.** `tmp/` is ignored so local scratch files do not pollute `git status`.
- **Launch materials.** Added release notes and launch-copy drafts in [`docs/release-notes-4.1.0.md`](./docs/release-notes-4.1.0.md) and [`docs/tweet-4.1.0.md`](./docs/tweet-4.1.0.md).
- **Versions.** `compose-agent` → `4.1.0`. `jetpack-compose-audit` → `4.1.0`.

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
