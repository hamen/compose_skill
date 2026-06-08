# Diagnostics

Copy-pasteable Gradle and code snippets the auditor can recommend (or run themselves) to back findings with measured evidence rather than source inference. Every snippet is anchored to an official source — cite the same URL in the report.

## 1. Compose Compiler Reports & Metrics

The single highest-leverage diagnostic. Generates per-composable skippability and per-class stability reports, plus aggregate metrics.

**Reference:** <https://developer.android.com/develop/ui/compose/performance/tooling>, <https://developer.android.com/develop/ui/compose/performance/stability/diagnose>

### Primary path — automatic (what the skill actually does)

The skill ships a Gradle init script at `scripts/compose-reports.init.gradle`. SKILL.md Step 4 runs it against the target without modifying any of the user's files:

```bash
cd <target> && ./gradlew <compile-task> \
    --init-script <skill-dir>/scripts/compose-reports.init.gradle \
    --no-daemon --quiet
```

The init script targets every module that applies the Compose Compiler plugin and writes reports to each module's `build/compose_audit/` directory. No `build.gradle.kts` edits are required on the target.

### Fallback path — manual edit (only when the init-script flow is blocked)

If the auditor (human or skill) cannot use `--init-script` — for example, a locked-down CI that rejects unknown init scripts — ask the user to add this block to the module's `build.gradle.kts`:

```kotlin
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_compiler")
    metricsDestination = layout.buildDirectory.dir("compose_compiler")
}
```

(Requires the Compose Compiler Gradle plugin, default since Kotlin 2.0. On older toolchains use `kotlinOptions.freeCompilerArgs += ["-P", "plugin:androidx.compose.compiler.plugins.kotlin:reportsDestination=..."]`.)

### Reading the output

Run a release-variant build, then inspect:

- `*-classes.txt` — stability inference per class (`stable` / `unstable` / `runtime`)
- `*-composables.txt` — per-composable `skippable` / `restartable` / `readonly` flags
- `*-composables.csv` — same data, machine-readable
- `*-module.json` — aggregate counts

**Use in the audit:** when a Performance or State finding alleges an unstable param or non-skippable composable, cite the relevant line of `*-classes.txt` or `*-composables.txt`. Without these reports, stability claims are *inferred* — say so explicitly in the report's Notes And Limits.

## 2. `compose_compiler_config.conf` — Marking Third-Party Types Stable

When unstable types come from modules without the Compose compiler (e.g. third-party data classes), mark them stable from outside.

**Reference:** <https://developer.android.com/develop/ui/compose/performance/stability/fix>

Create `compose_compiler_config.conf` at the project root (one fully-qualified class per line, glob patterns allowed):

```conf
# Mark third-party types stable so Compose can skip composables that take them
com.example.thirdparty.Money
com.example.thirdparty.User
com.example.thirdparty.events.*
java.time.*
```

Wire it into the module's `build.gradle.kts`:

```kotlin
composeCompiler {
    stabilityConfigurationFiles.add(
        rootProject.layout.projectDirectory.file("compose_compiler_config.conf")
    )
}
```

**Use in the audit:** if a project consumes third-party types in widely reused composables and skipping is broken, recommend a stability config file before recommending wrapper UI models.

## 3. Baseline Profile Module Skeleton

Improves cold start and frame timing by precompiling hot paths. The presence of a baseline profile module + `ProfileInstaller` in the consumer is a positive Performance signal.

**Reference:** <https://developer.android.com/develop/ui/compose/performance/baseline-profiles>, <https://developer.android.com/topic/performance/baselineprofiles/overview>

Module `:baselineprofile` (a `com.android.test` module) `build.gradle.kts`:

```kotlin
plugins {
    id("com.android.test")
    id("org.jetbrains.kotlin.android")
    id("androidx.baselineprofile")
}

android {
    targetProjectPath = ":app"
    defaultConfig { minSdk = 28 }
}

dependencies {
    implementation("androidx.test.ext:junit:1.2.1")
    implementation("androidx.test.uiautomator:uiautomator:2.3.0")
    implementation("androidx.benchmark:benchmark-macro-junit4:1.3.4")
}
```

Generator class:

```kotlin
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {
    @get:Rule val rule = BaselineProfileRule()

    @Test
    fun generate() = rule.collect(packageName = "com.example.app") {
        startActivityAndWait()
        // exercise the user-critical journey
    }
}
```

In the consumer (`:app`):

```kotlin
plugins {
    id("androidx.baselineprofile")
}

dependencies {
    "baselineProfile"(project(":baselineprofile"))
    implementation("androidx.profileinstaller:profileinstaller:1.4.1")
}
```

**Use in the audit:** check for a `baseline-prof.txt` artifact and a `ProfileInstaller` initializer. Their absence on a mature app is worth flagging; their presence is positive evidence.

## 4. R8 / Minify Hygiene

Compose performance assumes release-mode R8. Debug builds run unoptimized — never benchmark them.

**Reference:** <https://developer.android.com/develop/ui/compose/performance> ("Run in Release Mode with R8")

In `:app/build.gradle.kts`:

```kotlin
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro",
            )
        }
    }
}
```

**Use in the audit:** quick grep — `rg -n 'isMinifyEnabled' -g '*.gradle*'`. If the release block has `isMinifyEnabled = false`, that's a release-hygiene deduction on its own.

## 5. Strong Skipping Mode Confirmation

Determine Strong Skipping **per scored module**, not just once for the whole repo:

- Kotlin **2.0.20+** / Compose Compiler **1.5.4+**: Strong Skipping is **ON by default** unless the module explicitly opts out.
- Older Kotlin / compiler tracks: Strong Skipping is **OFF by default** unless the module explicitly opts in via `-Pandroidx.compose.compiler.enableStrongSkippingMode=true` or the equivalent `freeCompilerArgs` flag.
- If different modules disagree, record `Strong Skipping: mixed` in the report and name which modules ran with which table. When one overall Performance score covers both module types, explain the module mix and use the most conservative cap that the evidence supports.

**Reference:** <https://developer.android.com/develop/ui/compose/performance/stability/strongskipping>

Confirm the project's Kotlin version:

```bash
rg -n 'kotlin\s*=\s*"' -g '*.toml'
rg -n 'org\.jetbrains\.kotlin' -g '*.gradle*'
```

Then look for explicit Strong Skipping overrides:

```bash
rg -n 'enableStrongSkippingMode|androidx\.compose\.compiler\.enableStrongSkippingMode|StrongSkipping' -g '*.gradle*' -g '*.properties'
```

- On Kotlin **2.0.20+**, treat Strong Skipping as ON unless a module explicitly sets `enableStrongSkippingMode = false` (or the equivalent flag). Flag opt-outs and require justification.
- On older Kotlin tracks, treat Strong Skipping as OFF unless a module explicitly opts in. If you see an opt-in flag, apply the SSM-on rubric only to the affected modules.

## 6. Quick Triage Recipe

When you arrive at a Compose repo, run these in order before scoring:

1. `rg -n 'androidx\.compose' -g '*.gradle*' -g '*.toml'` — confirm Compose presence (fast-fail).
2. `rg -n 'kotlin\s*=\s*"' -g '*.toml'` and `rg -n 'org\.jetbrains\.kotlin' -g '*.gradle*'` — record Kotlin version (Strong Skipping baseline).
3. `rg -n 'enableStrongSkippingMode|androidx\.compose\.compiler\.enableStrongSkippingMode|StrongSkipping' -g '*.gradle*' -g '*.properties'` — explicit Strong Skipping opt-ins / opt-outs; note the module path for each hit.
4. `rg -n 'isMinifyEnabled' -g '*.gradle*'` — release hygiene.
5. Run Step 4 of SKILL.md — the init script generates compiler reports automatically. If the build fails, read any existing `composeCompiler { reportsDestination ... }` output the project already produces; otherwise note the fallback in the report.
6. `rg -l 'baselineProfile|ProfileInstaller' -g '*.gradle*' -g '*.kt'` — baseline-profile presence.

These six checks tell you what kind of evidence is available before any rubric-level reading.
