# KMP Rules

Rules for any Kotlin Multiplatform project. Read [`weekendware-rules.md`](weekendware-rules.md) first — universal rules apply here too. These rules extend, not replace, the meta file. After this, read any project-specific rules file if one exists.

---

## Folder Structure

| ID | Rule |
|----|------|
| FS-01 | Layer-based package structure in `commonMain` |
| FS-02 | Platform source sets mirror `commonMain` package paths for `expect`/`actual` |
| FS-03 | Platform source sets are thin — `expect`/`actual` and platform DI only |
| FS-04 | Drawables and fonts in `commonMain/composeResources/` — strings via single-source tool |
| FS-05 | SQLDelight `.sq` files follow the package path under `commonMain/sqldelight/` |
| FS-06 | Test structure mirrors source structure |
| FS-07 | Stay single-module until ~10 features — extract `core/` first, then feature modules |

### FS-01 · Layer-based package structure in `commonMain`

```
commonMain/kotlin/org/weekendware/<project>/
├── crash/          ← PHI scrubber, crash utilities
├── data/
│   ├── local/
│   │   └── database/   ← SQLDelight driver (expect declaration here)
│   ├── remote/         ← Ktor API clients
│   └── repository/     ← interfaces + implementations
├── di/             ← Koin modules
├── domain/
│   ├── model/      ← pure Kotlin domain models
│   └── usecase/    ← single-responsibility use cases
└── presentation/
    ├── components/ ← shared composables
    ├── theme/      ← Colors, Typography, Shapes, Spacing
    └── <feature>/  ← one folder per screen: Screen.kt, ViewModel.kt
```

> **Why:** Consistent layering makes every file locatable without searching. New engineers know where to look for a repository, a use case, or a ViewModel without reading the whole codebase.

*Source: Android Architecture Guide, Now in Android, JetBrains KMP docs*

---

### FS-02 · Platform source sets mirror `commonMain` package paths for `expect`/`actual`

```
commonMain/.../data/local/database/DatabaseDriverFactory.kt   ← expect
androidMain/.../data/local/database/DatabaseDriverFactory.kt  ← actual
iosMain/...   /data/local/database/DatabaseDriverFactory.kt   ← actual
desktopMain/.../data/local/database/DatabaseDriverFactory.kt  ← actual
```

> **Why:** The Kotlin compiler requires `actual` declarations to be in the same package as their `expect`. Mirroring the folder path enforces this and makes platform implementations easy to find.

> **Failure mode:** Platform implementation placed at a different package path. Kotlin throws a compile error on every target — looks unrelated to the file location.

---

### FS-03 · Platform source sets are thin — `expect`/`actual` and platform DI only

Business logic belongs in `commonMain`. Platform source sets contain only `expect`/`actual` implementations and platform-specific DI bindings.

```
androidMain/
├── data/local/database/DatabaseDriverFactory.kt  ← actual
└── di/PlatformModule.kt                          ← Android-specific DI bindings
```

> **Failure mode:** A use case added to `androidMain` because it "only makes sense on Android." Three months later someone tries to replicate the feature on Desktop and has to find and port it manually.

---

### FS-04 · Drawables and fonts in `commonMain/composeResources/` — strings via single-source tool

```
commonMain/composeResources/
├── drawable/   ← SVGs, vector assets
└── font/       ← font files
```

Compose Multiplatform's resource system generates type-safe accessors from `composeResources/`. Strings are managed via a generator tool (see CB-02) to maintain a single source of truth rather than manually syncing platform-specific files.

> **Failure mode:** A font or SVG placed in `androidMain/res/` compiles on Android. iOS and Desktop builds have no access to it and either crash or render nothing.

*Source: JetBrains Compose Multiplatform resources docs. For strings, see CB-02.*

---

### FS-05 · SQLDelight `.sq` files follow the package path under `commonMain/sqldelight/`

```
commonMain/sqldelight/org.weekendware.<project>/database/
├── User.sq
└── Message.sq
```

> **Why:** SQLDelight uses the folder path to determine the generated package name. Placing files in the wrong location generates code in the wrong package, breaking all imports.

*Source: SQLDelight official documentation*

---

### FS-06 · Test structure mirrors source structure

A test for `presentation/chat/ChatViewModel.kt` lives at `commonTest/presentation/chat/ChatViewModelTest.kt`.

```
commonTest/kotlin/org/weekendware/<project>/
├── data/repository/       ← repository fake + tests
├── domain/usecase/        ← use case tests
└── presentation/
    └── <feature>/         ← ViewModel tests
```

> **Failure mode:** Tests scattered at the top level of `commonTest`. Coverage gaps are invisible. A new feature is built with no tests and nobody notices because there's no clear place they should be.

---

### FS-07 · Stay single-module until ~10 features — extract `core/` first, then feature modules

Multi-module adds real Gradle complexity. A single `composeApp` module with convention plugins (L-02) builds fast and stays simple until the codebase is large enough that build times or team separation justify the overhead.

When the time comes: extract `core/` (theme, shared UI, utilities) first. Then extract feature modules one at a time. Never extract everything at once.

*Source: Lackner Chirp course (2026), Now in Android modularisation guide*

---

## Architecture

KMP-specific architecture rules. Universal rules live in `weekendware-rules.md`.

| ID | Rule |
|----|------|
| A-04 | No raw SQL outside `.sq` files |
| A-05 | Every screen has a `Screen` + `ScreenContent` composable split |
| A-06 | Every UI state has a `@Preview` — loading, empty, error, success minimum |
| A-07 | Platform differences via `expect`/`actual` — no runtime platform checks |
| A-11 | No Android framework types in `commonMain` |
| A-12 | `StateFlow` in ViewModels — `remember { mutableStateOf }` only for UI-local state |
| A-13 | UI state is a single `@Immutable data class` — no parallel `StateFlow` fields |
| A-14 | `modifier` is the first optional parameter on every composable |
| A-15 | New Koin module per feature domain |
| A-16 | Use `Modifier.Node` over `composed { }` for custom modifiers |
| A-17 | `clip` + `background` must come before `clickable` in modifier chains |
| A-18 | Use `mutableIntStateOf` / `mutableLongStateOf` for primitive state |

### A-04 · No raw SQL outside `.sq` files

No raw SQL strings in Kotlin. Use SQLDelight's generated type-safe API exclusively.

> **Why:** SQLDelight targets multiple platforms with different drivers. A raw SQL string in Kotlin bypasses the type-safe generated API and only works on whichever platform you tested.

> **Failure mode:** Table renamed in `.sq` — every caller caught at compile time. One raw string silently returns zero rows at runtime.

*Source: SQLDelight official documentation*

---

### A-05 · Every screen has a `Screen` + `ScreenContent` composable split

`Screen` calls `koinViewModel()` and observes `StateFlow`. `ScreenContent` is a pure composable that receives state and callbacks.

> **Why:** Android UI guide: *"Don't pass ViewModel instances down to other composables. Pass down just the state."* `koinViewModel()` can't be called in test or preview environments.

> **Failure mode:** Every `@Preview` or unit test on a composable that calls `koinViewModel()` crashes because Koin isn't initialised.

*Source: Android UI Architecture Guide*

---

### A-06 · Every UI state has a `@Preview` — loading, empty, error, success minimum

> **Why:** Compose Multiplatform previews on the JVM target are the only rapid visual feedback loop in a KMP project — there's no native iOS rendering in Android Studio.

> **Failure mode:** The avatar white-box bug went unreviewed until it shipped because there was no preview rendering that avatar state.

*Source: Compose Rules project, Android testing guide*

---

### A-07 · Platform differences via `expect`/`actual` — no runtime platform checks

Never use `if (isAndroid())` or similar runtime checks to branch behaviour. Use `expect`/`actual` instead.

> **Why:** `expect`/`actual` fails at compile time on any platform without an implementation. A runtime check compiles on all platforms but silently skips the iOS branch during Android testing.

> **Failure mode:** `DatabaseDriverFactory` via runtime check: iOS build compiles with a no-op database driver and produces zero errors. Data silently doesn't persist.

*Source: JetBrains KMP documentation*

---

### A-11 · No Android framework types in `commonMain`

No `android.content.Context`, `android.app.Activity`, or any other Android-only import in `commonMain`.

> **Why:** If a class in `commonMain` imports `android.content.Context`, it compiles on Android but fails with an unresolved reference on iOS and Desktop.

> **Failure mode:** Shared code that was "heavily Android-centric" — a known KMP pitfall. The iOS build breaks with zero explanation.

*Source: Android Architecture Guide, KMP pitfalls research 2025*

---

### A-12 · `StateFlow` in ViewModels — `remember { mutableStateOf }` only for UI-local state

ViewModels expose state via `StateFlow<UiState>`. `mutableStateOf` is only for composable-local ephemeral state (e.g. focus, expanded).

> **Why:** Swift interop works with `StateFlow` but not with Compose `State`. A ViewModel using `mutableStateOf` is unobservable from SwiftUI.

> **Failure mode:** Silent iOS-only breakage with no compile error. The Swift wrapper for the shared module simply can't observe the state.

*Source: KMP pitfalls research 2025*

---

### A-13 · UI state is a single `@Immutable data class` — no parallel `StateFlow` fields

Model state as a single sealed class or `@Immutable data class`. Do not use multiple `StateFlow` fields to represent what is logically one state.

> **Why:** Separate `isLoading: StateFlow<Boolean>` and `name: StateFlow<String>` can produce impossible combined states. A sealed `UiState` makes impossible states unrepresentable.

> **Failure mode:** Race condition: `isLoading = false` and `name = ""` simultaneously — an "empty success" indistinguishable from a legitimate empty state.

*Source: Now in Android architecture doc, Compose Rules project*

---

### A-14 · `modifier` is the first optional parameter on every composable

Parameter order: required params → `Modifier` (first optional) → other optional → trailing lambda.

> **Why:** Compose team convention enforced by Compose Rules linting.

> **Failure mode:** `modifier` placed last. A parent applies `fillMaxWidth()` — silently ignored because the internal layout never receives it.

*Source: Compose Rules project*

---

### A-15 · New Koin module per feature domain

If a feature has its own repository, use case, and ViewModel, it gets its own Koin module. Don't bloat `sharedModule`.

> **Failure mode:** `sharedModule` grows to 40 bindings. A single missing definition causes a runtime `NoDefinitionFoundException` that's hard to trace.

---

### A-16 · Use `Modifier.Node` over `composed { }` for custom modifiers

> **Why:** `composed { }` runs the modifier factory on every recomposition, allocating a new lambda each time. `Modifier.Node` is allocated once and reused across recompositions.

> **Failure mode:** A custom ripple or animation modifier using `composed { }` inside a list item. Every scroll recomposition allocates a new modifier instance. Visible in profiling as excess allocations on a smooth-scroll benchmark.

*Source: Nacho Lopez / compose-rules project (ktlint 1.8 / Kotlin 2.2.21, 2025–26)*

---

### A-17 · `clip` + `background` must come before `clickable` in modifier chains

```kotlin
Modifier
    .clip(RoundedCornerShape(12.dp))
    .background(color)
    .clickable { ... }
```

> **Why:** The ripple effect is drawn in modifier application order. If `clickable` comes before `clip`, the ripple bleeds outside the shape boundary on every interaction.

> **Failure mode:** A rounded card with `clickable` before `clip`. Tap anywhere on the card — the ripple spills outside the rounded corners into the surrounding layout.

*Source: Nacho Lopez / compose-rules project (2025–26)*

---

### A-18 · Use `mutableIntStateOf` / `mutableLongStateOf` for primitive state

> **Why:** `mutableStateOf<Int>` boxes the primitive on every read and write. The specialised variants store the value unboxed.

> **Failure mode:** A scroll position or counter stored as `mutableStateOf<Int>`. Updated on every frame. Allocates an `Integer` object per update — invisible in unit tests, shows up as GC pressure in a production frame profiler.

*Source: Nacho Lopez / compose-rules project (2025–26)*

---

## Build & DevOps

KMP-specific build rules. Universal build rules live in `weekendware-rules.md`.

| ID | Rule |
|----|------|
| B-01 | All versions in `libs.versions.toml` — no version numbers in `build.gradle.kts` |
| B-02 | Configuration cache and parallel builds enabled in `gradle.properties` |
| B-03 | JDK version as a repo variable — never hardcoded in workflow YAML |
| B-05 | Pin macOS runner and Xcode version |
| B-11 | Add `kotlinx-collections-immutable` — use `ImmutableList`/`ImmutableMap` for Compose params |
| B-12 | Enable Compose compiler stability reports in CI to catch unstable params |

### B-01 · All versions in `libs.versions.toml` — no version numbers in `build.gradle.kts`

> **Why:** Version catalogs are the only way to share dependency versions across `commonMain`, `androidMain`, and `iosMain` with IDE autocompletion.

> **Failure mode:** A library bumped in one source set but not another. Silent ABI mismatch that compiles but crashes at runtime.

*Source: Android docs (Migrate to version catalogs), Gradle docs*

---

### B-02 · Configuration cache and parallel builds enabled in `gradle.properties`

```properties
org.gradle.jvmargs=-Xmx4096M -Dfile.encoding=UTF-8
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true
ksp.incremental=true
```

Enables parallel task execution across KMP modules. Research cites 70% CI compilation reduction.

*Source: JetBrains KMP GitHub Actions docs, mvpfactory.io*

---

### B-03 · JDK version as a repo variable — never hardcoded in workflow YAML

`JAVA_JDK_VERSION = 21` as a GitHub repo variable. Already hit: JDK 26 breaks the Kotlin compiler in our Gradle version.

*Source: kmpship.app CI/CD guide*

---

### B-05 · Pin macOS runner and Xcode version

`runs-on: macos-14` + `XCODE_VERSION` as a repo variable. GitHub updates macOS images on their own schedule — an Xcode major version bump can silently break framework compilation.

*Source: kmpship.app CI/CD guide*

---

### B-11 · Add `kotlinx-collections-immutable` — use `ImmutableList`/`ImmutableMap` for Compose params

The Compose compiler treats standard `List` and `Map` as unstable (they're mutable interfaces). Passing them as composable params forces full recomposition even when the contents haven't changed. `ImmutableList`/`ImmutableMap` from `kotlinx-collections-immutable` are inferred as stable.

> **Failure mode:** A composable receives `List<ChatMessage>`. A parent recomposes for an unrelated reason. The composable recomposes too, even though the list is identical. Enable compose compiler reports (B-12) to surface these.

*Source: Nacho Lopez / compose-rules project (2025–26), JetBrains KMP docs*

---

### B-12 · Enable Compose compiler stability reports in CI to catch unstable params

The compiler outputs a `composables.txt` report listing every composable that is non-restartable or non-skippable due to unstable parameters.

```kotlin
// composeApp/build.gradle.kts
composeCompiler {
  reportsDestination = layout.buildDirectory.dir("compose_reports")
  metricsDestination = layout.buildDirectory.dir("compose_reports")
}
```

*Source: Jake Wharton talks (2025–26), Jetpack Compose compiler docs*

---

## QA & Testing

KMP-specific testing rules. Universal QA rules live in `weekendware-rules.md`.

| ID | Rule |
|----|------|
| QA-01 | ViewModel tests use `runTest` + Turbine — no `delay()` assertions |
| QA-03 | Business logic tests in `commonTest` — platform-specific only for `expect`/`actual` |
| QA-04 | Compose UI tests use `runComposeUiTest` (v2) with `testTag` — never text-based selectors |
| QA-05 | `StandardTestDispatcher` is the default — call `advanceUntilIdle()` after `LaunchedEffect` |
| QA-07 | SQLDelight tests use in-memory drivers only — never file-based |
| QA-08 | Use `waitUntil { }` over idling resources for async UI test conditions |

### QA-01 · ViewModel tests use `runTest` + Turbine — no `delay()` assertions

Turbine is KMP-compatible and runs in `commonTest`. `StateFlow` always replays its current value to new collectors, so `awaitItem()` catches initialisation bugs invisible to delay-based tests.

> **Failure mode:** `delay(100)` then assert. Passes on a fast CI machine. Fails randomly on a slower one.

*Source: Android Developers (testing flows), kmpship.app 2025*

---

### QA-03 · Business logic tests in `commonTest` — platform-specific only for `expect`/`actual`

Tests in `commonTest` execute on every configured target automatically. `commonTest` coverage is the only cross-platform safety net.

> **Failure mode:** A repository test in `desktopTest`. The iOS SQLite driver has a subtle NULL handling difference. Passes on Desktop. Ships on iOS as a bug.

*Source: kotlinlang.org multiplatform testing docs*

---

### QA-04 · Compose UI tests use `runComposeUiTest` (v2) with `testTag` — never text-based selectors

CMP 1.11.0 (May 2026) unified the Compose test API across all targets. Use `onNodeWithTag()` exclusively.

> **Failure mode:** `onNodeWithText("Sign In")` passes. Copy changes to "Log In". Selector finds nothing. Looks like a regression — isn't one.

*Source: JetBrains Blog May 2026 (CMP 1.11.0)*

---

### QA-05 · `StandardTestDispatcher` is the default — call `advanceUntilIdle()` after `LaunchedEffect`

CMP 1.11.0 changed the default test dispatcher. Coroutines no longer run automatically — they queue. Any composable with a `LaunchedEffect` requires explicit scheduler advancement before asserting.

> **Failure mode:** Test asserts on loaded state immediately after setting content. Effect hasn't run. Asserts the loading state, or a lazy fix wraps everything in `UnconfinedTestDispatcher`, hiding real concurrency bugs.

*Source: JetBrains Blog May 2026 (CMP 1.11.0)*

---

### QA-07 · SQLDelight tests use in-memory drivers only — never file-based

File-backed test databases persist between runs on Desktop, creating ordering dependencies. In-memory databases are hermetic and parallelisable.

> **Failure mode:** Test A inserts a user. Test B expects one result. File-backed database persists. Test B sees two. Suite is order-dependent and fails randomly in CI.

*Source: SQLDelight official docs, kmpship.app 2025*

---

### QA-08 · Use `waitUntil { }` over idling resources for async UI test conditions

`waitUntil { condition }` polls the semantics tree directly with a configurable timeout — no production code hooks, no coupling between tests and app internals.

> **Failure mode:** An idling resource registered in the ViewModel. Fails silently when the ViewModel is replaced during a refactor and the idling resource hook is forgotten. `waitUntil { onNodeWithTag("chat-list").isDisplayed() }` needs no hooks — it just waits for the node to appear.

*Source: Jake Wharton (2025–26), Compose UI testing docs*

---

## Security

KMP-specific security rules. Universal security rules live in `weekendware-rules.md`.

| ID | Rule | Level |
|----|------|-------|
| S-09 | At-rest database encryption before real users | **Blocker** |
| S-10 | Secure token storage — Android Keystore, iOS Keychain, never SharedPreferences | **Blocker** |
| S-11 | Desktop target has no secure keystore — document the limitation explicitly | Document |

### S-09 · At-rest database encryption before real users `BLOCKER`

SQLCipher (KmpSqlencrypt) with an encryption key stored in Android Keystore. Key is never hardcoded.

> **Failure mode:** A rooted device or forensic backup exposes all stored data in plaintext.

*Source: OWASP Mobile M9, KmpSqlencrypt GitHub*

---

### S-10 · Secure token storage — Android Keystore, iOS Keychain, never SharedPreferences `BLOCKER`

Auth tokens must not land in `SharedPreferences` or `UserDefaults`. `androidx.security:security-crypto` deprecated July 2025 — use KVault or Kassaforte via `expect`/`actual`.

> **Failure mode:** JWT in `SharedPreferences`: readable without root via content provider on some Android versions.

*Source: OWASP Mobile M9, KVault GitHub, Kassaforte (May 2026)*

---

### S-11 · Desktop target has no secure keystore — document the limitation explicitly

Android has Keystore; iOS has Keychain. The JVM Desktop target has neither. Either implement OS-level credential storage or document that Desktop has reduced security guarantees. Never fall back silently to a plaintext file.

---

## Community 2025–26

Patterns sourced from Philipp Lackner's 2026 GitHub activity, Jake Wharton's blog (Nov–Apr 2025–26), and Chris Banes' blog (Feb 2025).

| ID | Rule |
|----|------|
| L-01 | Extract UI interaction state into plain state holder classes |
| L-02 | Shared build configuration in `build-logic/` convention plugins |
| L-03 | Use `value class` for domain identifiers |
| L-04 | Use `Stylable` + `Style { }` for reusable visual styling — not modifier chains |
| JW-01 | Don't use the Compose BOM — declare group versions directly in the version catalog |
| JW-02 | Use AndroidX betas — don't wait for stable releases |
| JW-03 | Remove all `-ktx` dependency suffixes |
| CB-01 | Don't default to `Sequence` for performance — benchmark first |
| CB-02 | Use a single-source string tool for multiplatform |

### L-01 · Extract UI interaction state into plain state holder classes

ViewModels own business-logic state via `StateFlow`. Form state, field validation, step tracking, and focus go in a plain Kotlin class — not the ViewModel.

> **Why:** Android docs: *"When the amount of state increases, delegate to state holder classes."*

> **Failure mode:** A ViewModel with 15+ state fields for a multi-step onboarding flow — impossible to test individual steps in isolation.

*Source: Lackner AndroidStateManagement-CustomStateHolders (Jun 2026), Android docs*

---

### L-02 · Shared build configuration in `build-logic/` convention plugins

Convention plugins replace copy-pasted Gradle config across modules. Same pattern used in every Lackner course from Runique onwards, and Now in Android.

> **Failure mode:** Updating `compileSdk` requires touching every module's `build.gradle.kts`. One module gets a different `jvmTarget` — iOS framework build fails in CI only.

*Source: Lackner Chirp (2026 KMP course), Now in Android*

---

### L-03 · Use `value class` for domain identifiers

The K2 compiler makes `value class UserId(val id: String)` fully reliable in 2026. Prevents passing a `ConversationId` where a `UserId` is expected — compile-time enforcement of domain type safety.

> **Failure mode:** Raw `String` IDs: a `userId` is passed to a function expecting a `messageId`. Compiles. Fails at runtime when the query returns nothing.

*Source: Lackner kotlin-beyond-syntax (Feb 2026)*

---

### L-04 · Use `Stylable` + `Style { }` for reusable visual styling — not modifier chains

The Compose Styles API (2026, still experimental) packages a full set of visual attributes into a single `Style` object. Styles inherit down the composable tree, last-write-wins, and support animated state transitions for `press`, `hover`, `focused`, and `disabled` without manual `LaunchedEffect` wiring.

**Pattern:** Always initialise the incoming style parameter as `Style {}` (empty). Apply your internal design-system style first, then chain the caller's override: `baseButtonStyle.then(style)`.

```kotlin
val filledButton = baseButton.then(Style {
    background(AppColors.brand)
    contentColor(Color.White)
    press { animate { background(AppColors.brandPressed) } }
    disabled { background(Color.Gray); contentColor(Color.White) }
})

Box(Modifier.stylable(styleState, filledButton)) { ... }
```

**Note:** Still experimental — requires `ComposeFou­ndationFlags.isInheritedTextStyleEnabled = true` in `MainActivity`. Use on new design-system components; do not refactor existing modifiers until the API is stable.

*Source: Lackner — "Jetpack Compose Styles API" (2026)*

---

### JW-01 · Don't use the Compose BOM — declare group versions directly in the version catalog

Gradle auto-aligns sibling artifacts within a library group via module metadata — the BOM's core job is already handled. The Compose BOM ships inconsistently, masks actual library versions behind a date string, and gets partially overridden the moment you adopt AndroidX betas.

*Source: Jake Wharton, "Let's defuse the Compose BOM" (Dec 2025)*

---

### JW-02 · Use AndroidX betas — don't wait for stable releases

AndroidX API is locked at `beta01` — all subsequent beta and RC releases are bug-fixes only, no breaking changes. Google's first-party apps test these before public release. Waiting for stable means months of exposure to known bugs that are already fixed in beta. Requires adequate test coverage before adopting.

*Source: Jake Wharton, "You should use AndroidX betas" (Nov 2025)*

---

### JW-03 · Remove all `-ktx` dependency suffixes

Android KTX extension modules are being officially deprecated — their Kotlin extensions have been absorbed into the core AndroidX libraries. Keeping `-ktx` dependencies adds a redundant artifact that will not receive future updates. Use Android Lint to surface remaining `-ktx` imports.

*Source: Jake Wharton, "An update on Android KTX" (April 2026)*

---

### CB-01 · Don't default to `Sequence` for performance — benchmark first

kotlinx-benchmark results show `List` outperforms `Sequence` for typical-sized collections. `Sequence` carries per-element lambda allocation overhead that only pays off with large collections and multiple chained operations. Blindly swapping `List` for `Sequence` is premature optimisation that often makes things slower.

*Source: Chris Banes, "Should you use Kotlin Sequences for Performance?" (Feb 2025)*

---

### CB-02 · Use a single-source string tool for multiplatform

Don't manually maintain platform-specific string files (`strings.xml` and `Localizable.strings`). Use a tool — Twine, Lyricist, or Moko Resources — to generate all platform strings from a single YAML or Kotlin source of truth.

> **Why:** Manual sync produces copy drift and missed translations.

*Source: Chris Banes, "Multiplatform Strings" (Feb 2025)*
