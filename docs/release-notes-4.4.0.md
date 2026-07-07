# Release notes — 4.4.0 (2026-07-07)

**`compose-agent` only.** `jetpack-compose-audit` unchanged at `4.3.2`.

Closes the last follow-up flagged across the 4.3.x cross-reviews: `compose-agent` had no evals of its own, so its authoring rules were only ever exercised through the *audit's* acceptance cases (which grade an audit of existing code, not code the skill writes).

## New: write-mode acceptance evals

`skills/compose-agent/evals/evals.json` — five cases that hand the skill a *writing* task and grade the Compose it produces:

- **Case 0 — cross-phase back-write avoidance.** Align a value to a label's measured width without writing `onSizeChanged` state read in composition; expects a layout-phase solution.
- **Case 1 — Strong Skipping false leads.** Make a row skip under SSM; expects *no* `remember`-wrapped callbacks and *no* `remember(index)` on pure expressions — real levers only (stable types, keys, verified with recomposition counts).
- **Case 2 — snapshot self-invalidation avoidance.** Build a per-entry lookup with `remember(entries)`, not by mutating a `mutableStateMapOf` during composition.
- **Case 3 — phase-correct reads.** Feed a per-frame `Float` through a `() -> Float` provider or a layout/draw-phase read, keeping composition stable.
- **Case 4 — lifecycle-aware collection.** Collect a `StateFlow` with `collectAsStateWithLifecycle()`, side-effect free, modifier-hygienic.

Cases 0–2 mirror the `jetpack-compose-audit` scoring rules 1:1, so the authoring path and the audit path stay in lockstep.

## CI enforcement

`bin/ci` now:

- structurally validates **every** `skills/<name>/evals/evals.json` — valid JSON, unique numeric ids, non-empty `prompt` / `expected_output`, `files` array, non-empty `expectations`;
- asserts the compose-agent suite keeps covering the three 4.3.x authoring rules (cross-phase back-write, Strong Skipping false lead, snapshot self-invalidation), so a future edit can't quietly drop coverage.

## Note

These evals are acceptance data, not a runner — run a model with the skill loaded against each prompt and check every expectation, the same way `jetpack-compose-audit/evals/evals.json` is used.

## Versions

`compose-agent` → **4.4.0** (SKILL + both `plugin.json`). `jetpack-compose-audit` stays at `4.3.2`.
