# Report Template

Write the audit report to `COMPOSE-AUDIT-REPORT.md` using this structure.

**Citation rule:** every finding (Critical Findings *and* per-category Evidence bullets) must include a `References:` line with at least one URL pointing to the official documentation rule the code violates. Use the URLs in `references/canonical-sources.md` and `references/scoring.md`. A finding without a citation should not appear in the report — that's the credibility lever this audit relies on.

```markdown
# Jetpack Compose Audit Report

Target: [repo path or module path]
Date: [YYYY-MM-DD]
Scope: [modules or directories audited]
Excluded from scoring: [paths or globs treated as samples / tests / previews]
Confidence: [High | Medium | Low]
Overall Score: [X/100]

## Scorecard

| Category | Score | Weight | Status | Notes |
|----------|-------|--------|--------|-------|
| Performance | X/10 | 35% | [fail / needs work / solid / excellent] | [short note] |
| State management | X/10 | 25% | [fail / needs work / solid / excellent] | [short note] |
| Side effects | X/10 | 20% | [fail / needs work / solid / excellent] | [short note] |
| Composable API quality | X/10 | 20% | [fail / needs work / solid / excellent] | [short note] |

## Critical Findings

List the most important findings first. Each finding should include:

- severity
- why it matters
- 2-4 concrete file examples
- the likely fix direction

Example format:

1. **Performance: repeated expensive work happens inside composition**
   - Why it matters: [brief reason]
   - Evidence: `path/a.kt:42`, `path/b.kt:117`
   - Fix direction: [brief recommendation]
   - References: <https://developer.android.com/develop/ui/compose/performance/bestpractices>

## Adjacent Findings

Use this section for concrete, user-visible problems the audit intentionally tracks outside the four numeric Compose categories. Do not change the 0-100 score for these findings, but do include them in `Critical Findings` or `Prioritized Fixes` when they are among the highest-leverage fixes.

### Android Launch UX

- Android 12+ splash icon status: [clean / risky / not configured / not inspected]
- Evidence: [`windowSplashScreenAnimatedIcon` theme item, resolved drawable file, `drawable-v31` override status]
- Finding, if risky: **Android 12+ static splash icon may render blurry** because the API 31+ resource resolves to a static `<vector>` / `<adaptive-icon>` / bitmap / layer-list instead of an `<animated-vector>` wrapper.
- Fix direction: keep the theme resource name stable and make it resolve to an `<animated-vector>` wrapper around the real vector on API 31+; no animators are required (an empty `<animated-vector>` is enough). The wrapper must reference a separately-named vector to avoid a self-reference loop.
- References: <https://developer.android.com/develop/ui/views/launch/splash-screen>, <https://developer.android.com/reference/androidx/core/splashscreen/SplashScreen>, <https://issuetracker.google.com/issues/520672537>

## Category Details

### Performance — [X/10]

**Ceiling check**

- Strong Skipping: [on / off / mixed across modules]
- Ceiling table applied: [SSM-on / SSM-off / mixed, with a one-line explanation]
- Module-wide `skippable%`: [x/y = z%]
- Named-only `skippable%`: [x/y = z% — state explicitly if this was the binding ceiling metric because zero-arg lambdas anchored the module-wide number]
- Unstable shared types from compiler: [count, or `n/a` if diagnostics unavailable]
- SSM-on binding evidence: [instance-recreation churn / expensive or broken `equals()` / unjustified opt-outs / none observed]
- Qualitative score: [X/10]
- Ceiling: [none / cap at N]
- Applied score: [X/10]

**What is working**

- [positive evidence]

**What is hurting the score**

- [problem 1]
- [problem 2]

**Animation performance signals**

- Status: [clean / risky / not present / not inspected]
- If risky, state exactly which animation rule affected Performance: composition-phase animated reads, `Animatable` not remembered, `rememberInfiniteTransition` kept alive offscreen, deprecated `animateItemPlacement()`, or missing lazy-list keys for `animateItem()`.
- Keep animation-driving mistakes in Side Effects instead when the root cause is launching animation work from composition or using `rememberCoroutineScope().launch { animateTo(...) }` for target-driven state changes.

**Paging list signals**

- Status: [clean / risky / not present / not inspected]
- If risky, state exactly which paging rule affected Performance: missing or index-only keys on `LazyPagingItems`, `itemSnapshotList` primary UI wiring, or paginated-stream duplicate-key risk.
- Keep ignored `LoadState` / duplicate loading flags in State Management instead (**Paging load-state handling**).
- Keep unconditional `refresh()` / `retry()` from composition in Side Effects.

**Evidence**

- `path/to/file1.kt:LL` — [brief reason] · References: <https://developer.android.com/...>
- `path/to/file2.kt:LL` — [brief reason] · References: <https://developer.android.com/...>

### State Management — [X/10]

**What is working**

- [positive evidence]

**What is hurting the score**

- [problem 1]
- [problem 2]

**Paging load-state signals**

- Status: [clean / risky / not present / not inspected]
- If risky, state exactly which paging rule affected State: ignored `LoadState.Error`, missing empty state, stuck refresh spinner, or duplicate manual loading flag shadowing `loadState`.
- Keep missing keys / index keys on `LazyPagingItems` in Performance (**Paging list correctness**) instead.

**Evidence**

- `path/to/file1.kt:LL` — [brief reason] · References: <https://developer.android.com/...>
- `path/to/file2.kt:LL` — [brief reason] · References: <https://developer.android.com/...>

### Side Effects — [X/10]

**What is working**

- [positive evidence]

**What is hurting the score**

- [problem 1]
- [problem 2]

**Animation side-effect signals**

- Status: [clean / risky / not present / not inspected]
- If risky, name the Side Effects issue explicitly: `Animatable.animateTo()` from composition, `rememberCoroutineScope().launch { animateTo(...) }` reacting to state instead of `LaunchedEffect(target)`, or `Animatable` + `LaunchedEffect` where `animate*AsState` / `updateTransition` would suffice.

**Evidence**

- `path/to/file1.kt:LL` — [brief reason] · References: <https://developer.android.com/...>
- `path/to/file2.kt:LL` — [brief reason] · References: <https://developer.android.com/...>

### Composable API Quality — [X/10]

**What is working**

- [positive evidence]

**What is hurting the score**

- [problem 1]
- [problem 2]

**Evidence**

- `path/to/file1.kt:LL` — [brief reason] · References: <https://developer.android.com/...>
- `path/to/file2.kt:LL` — [brief reason] · References: <https://developer.android.com/...>

## Prioritized Fixes

1. [Highest leverage fix]
2. [Second fix]
3. [Third fix]
4. [Optional follow-up]

## Notes And Limits

- [state if only part of the repo was audited]
- [state if confidence is medium/low]
- [state if some categories had limited surface area]
- Adjacent coverage notes: [UI tests / screenshot tests / focus-keyboard / KMP-CMP surfaces observed; state `none observed` if absent]
- Android Launch UX resources: [clean / risky / not configured / not inspected; include the splash icon resource path when configured]
- Strong Skipping mode: [on / off / mixed across modules; note any explicit module-level opt-in / opt-out]
- Weight choice: [default 35/25/20/20, or note any deviation and why]
- Renormalization: [list any N/A categories and the renormalized weights]
- Compiler diagnostics used: [yes / no — link to the Compose Compiler reports if generated; if yes, note which modules contributed, whether module-wide and named-only `skippable%` were available, and which ceiling table was applied; "no" means stability claims are inferred from source, not measured]

## Suggested Follow-Up

- Run `material-3` audit if the repo also shows likely design-system or Material 3 problems.
- Run `compose-agent focus on testing` if UI tests, screenshot tests, previews, or fake-image/platform-service patterns need deeper review.
- Run `compose-agent focus on focus` if keyboard, D-pad, TV, desktop, ChromeOS, or focus-restoration behavior is present.
- Run `compose-agent focus on kmp` if Compose Multiplatform, KMP source sets, `expect` / `actual`, or platform interop boundaries are present.
- Run `compose-agent focus on animation` if animation findings need a file-scope rewrite rather than an audit-level diagnosis.
- Run `compose-agent focus on paging` if paging list / load-state findings need a file-scope rewrite rather than an audit-level diagnosis.
```

## Tone

- Keep the tone strict and direct.
- Avoid filler praise.
- Give credit only where the codebase actually demonstrates good patterns.
- Prefer a few strong findings over dozens of weak bullets.
