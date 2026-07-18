# WeekendWare Rules

Universal standards that apply to every WeekendWare project. Read this file first, then determine your platform and read the platform rules file (e.g. `kmp-rules.md`), then read any project-specific rules file (`projects/<project>/rules/<project>-rules.md`).

---

## Pre-launch blockers

These must pass before any real user traffic reaches the app.

| Rule |
|------|
| API keys stay on the server — never in the binary or client config |
| No credentials, tokens, or user IDs in any log output |
| Session timeout after inactivity — 15 minutes, configurable |
| HTTPS enforced on staging and prod — no plaintext traffic |
| Every request to the API authenticated — no anonymous endpoints |
| Server-side rate limiting — independent of client UX |
| No request bodies logged on the server |

---

## Git

All eight rules are universal — they apply to every WeekendWare repo.

| Rule |
|------|
| Three long-lived branches only: `main`, `staging`, `develop` |
| Feature branches cut from `develop` only |
| Branch naming: `MMDDYYYY-description` |
| PRs always target `develop` — pass `--base develop` explicitly |
| No direct commits to `main` or `staging` |
| No AI attribution or co-author lines in commit messages |
| Update `README.md` before pushing to GitHub |
| Every project repo follows identical branching and commit standards |
| Feature PRs must attach the approved wireframe and mockup for the feature |

### Three long-lived branches only: `main`, `staging`, `develop`

`main` is production. `staging` is pre-release. `develop` is active development. Feature branches are cut from `develop`, named `MMDDYYYY-description`, and merged back to `develop` via PR. No other long-lived branches.

> **Why:** Short-lived feature branches prevent divergence. Every branch has a clear purpose and a clear merge target.

> **Failure mode:** A long-lived `feature/auth` branch diverges from `develop` for three weeks. Merge conflict hell. Work is duplicated or lost.

---

### Feature branches cut from `develop` — never from `main` or `staging`

> **Why:** `develop` is always ahead of `main`. A branch cut from `main` is missing everything that has shipped to develop since the last release.

> **Failure mode:** Feature cut from `main`, merged to `develop`. Silent regression — work already in develop is overwritten by the older base.

---

### Branch naming: `MMDDYYYY-description`

Date-first naming makes branches sort chronologically. Description is kebab-case and concise.

```
07162026-check-in-system
07162026-onboarding-flow
```

> **Failure mode:** `feature/auth`, `fix-bug`, `gavin-test` — no date, no context, no way to sort or prune stale branches.

---

### PRs always target `develop` — pass `--base develop` explicitly

```sh
gh pr create --base develop --title "..." --body "..."
```

> **Why:** GitHub defaults to the repo's default branch. If `main` is the default, a PR without `--base develop` silently targets production.

> **Failure mode:** Half-finished feature merged directly to `main`. It's in production. Rolling it back requires a revert PR and a hotfix cycle.

---

### No direct commits to `main` or `staging`

`main` only merges from `staging` at release time. `staging` only merges from `develop`. Direct commits bypass all CI checks and review.

> **Failure mode:** A "quick hotfix" committed directly to `main`. CI never ran. The fix introduced a regression that wouldn't have survived a basic compile check.

---

### No AI attribution or co-author lines in commit messages

Commit history is a professional artefact. Co-author lines for AI tools add noise and don't represent authorship in any legally or professionally meaningful sense.

---

### Update `README.md` before pushing to GitHub

The README is the public face of the repo. New agent hired? New folder? Rule changed? Update it. The README must always match what is actually in the repo.

> **Failure mode:** Logging infrastructure deleted. README still says "logging app". Anyone reading the repo sees a description that has nothing to do with the actual product.

---

### Every project repo follows identical branching and commit standards

Same three-branch structure, same naming convention, same PR-to-develop workflow — whether it's a KMP app, a Rust API, or a future web project. No repo gets a special exception.

---

### Feature PRs must attach the approved wireframe and mockup for the feature

Every feature PR must embed the wireframe and final mockup directly in the PR body as inline images — no links. Export PNGs from the HTML mockup files before opening the PR. GitHub renders inline images natively; a reviewer should see the approved design without leaving the PR.

Applies to both engineering PRs (opened by the Architect) and design PRs (opened by the Head of Design).

> **Why:** Code review without the design is an implementation check, not a product check. Inline images let Gavin verify at a glance that what was built matches what was approved.

> **Failure mode:** A feature ships with a layout that drifted from the approved design. Nobody caught it in review because the mockup wasn't visible in the PR.

---

## Architecture

Universal rules apply to any layered project on any platform.

| Rule |
|------|
| Repository interface in DI, never the concrete class |
| Build order: storage → service → API layer |
| `Result<T>` across layer boundaries — no thrown exceptions |
| Design artefact before code — no screen ships without an approved design |
| Screen design ships as two files — shell mockup (PR images) and states reference (builder + QA spec) |
| Design tokens before components — no hardcoded values in UI code |
| Empty states and error states are designed — first-class screens |
| Interactive element states designed — default, pressed, focused, disabled minimum |
| All UI meets WCAG AA contrast — 3:1 for components and icons, 4.5:1 for body text |
| Color is not the only means of conveying interactive state — WCAG 1.4.1 |
| Minimum touch target — 24×24px CSS (WCAG 2.5.8); 48dp Android, 44pt iOS |
| All code documented using the platform-appropriate documentation library |

### Repository interface in DI, never the concrete class

Bind and inject the interface. Never import the concrete implementation directly in a ViewModel or use case.

> **Why:** Android Data Layer guide: *"Relying on interfaces makes API implementations swappable."* In KMP, a ViewModel importing a concrete repository directly can't compile on a platform where that driver doesn't exist.

> **Failure mode:** No test doubles without mocking frameworks. Without interfaces, every test requires a real database or network.

*Source: Android Data Layer Guide*

---

### Build order: storage → service → API layer

Write the schema first, then service logic, then API handlers.

> **Why:** Writing service logic before the schema exists means writing against a moving target. Storage is the ground truth — everything above it depends on it.

> **Failure mode:** A service function depends on a column that doesn't exist yet. Compiles fine — until schema generation runs and throws.

*Source: Android Architecture Guide — three-layer dependency direction*

---

### `Result<T>` across layer boundaries — no thrown exceptions

Repositories return `Result<T>`. Use cases propagate it. ViewModels map it to UI state. Never throw across a layer boundary.

> **Why:** Android Data Layer guide: *"Use a `Result` class. This pattern models errors, making the UI aware of known errors."* In KMP, Kotlin exceptions on iOS cause hard crashes with stripped stack traces.

> **Failure mode:** A repository throws `NetworkException`. ViewModel doesn't catch it. Android: crash. iOS: `UnhandledException` with no stack trace. `Result<T>` forces callers to handle failure at compile time.

*Source: Android Data Layer Guide*

---

### Design artefact before code — no screen ships without an approved design

A Figma file, HTML mockup, or written design spec anchors every target platform to the same visual contract before any implementation begins.

> **Why:** Building without one produces divergent implementations that are expensive to fix.

> **Failure mode:** `SettingsScreen` BG unit toggle was implemented without a design. When removed, there was no artefact to verify the remaining sections against.

*Source: Material Design 3, Figma design system documentation*

---

### Screen design ships as two files — shell mockup and states reference

Every screen has exactly two design artefacts. Both must exist and be approved before a builder writes a line of code.

| File | What it shows | Used by |
|------|--------------|---------|
| `mockup-[feature]-v[n].html` | Full-screen render in all schemes or themes | PR body images, visual sign-off |
| `states-[feature]-v[n].html` | Every interactive element in every state | Builders (implementation spec), QA (test spec) |

The shell mockup answers "what does it look like?" The states reference answers "how does every element behave?" States are labelled (NAV-01, SEND-02, etc.) so QA can reference them directly in test descriptions and assertions.

Builders pull both files before implementing. QA writes tests against the states file — each labelled state maps to at least one automated test.

> **Why:** Without the states file, builders guess at inactive, pressed, loading, and error treatment every time — inconsistently, across every feature.

> **Failure mode:** Send button always active because no inactive/disabled state was designed. Users send empty messages. Server receives them. Onboarding logic breaks silently.

---

### Design tokens before components — no hardcoded values in UI code

All colours, spacing, and typography reference tokens from the design system. Never hardcode `Color(0xFF546857)` or `16.dp` inline.

> **Why:** M3: *"Design tokens are the building blocks of all UI elements, with the same tokens used in designs, tools, and code."* Hardcoded colours bypass `MaterialTheme.colorScheme` and the design system entirely.

> **Failure mode:** Dark mode. Any hardcoded colour is invisible to the theme system and renders identically in both modes — usually unreadable in one.

*Source: Material Design 3*

---

### Empty states and error states are designed — first-class screens

Every screen that can be empty or fail must have an explicit empty state and error state defined in the design and implemented in code.

> **Why:** Android architecture guide: *"UI elements handle all possible states."* The first time a user opens a new account, every screen is empty. That's not an edge case — it's the first impression.

> **Failure mode:** After the logging cleanup, `DashboardScreen` was a blank white rectangle with no indication of what it's for or what to do.

*Source: Android Architecture Guide, Now in Android sealed UI state pattern*

---

### Interactive element states designed — default, pressed, focused, disabled minimum

Every interactive element — button, icon button, nav item, input field — must have its visual states defined in the design before any implementation begins. Not every element has every state; a static icon has no pressed state. Use judgment. But every tappable element needs at minimum: default, pressed, and disabled.

| State | What it is |
|-------|-----------|
| Default | Unpressed, unfocused. |
| Focused / active | Input field selected, button receiving keyboard focus. |
| Pressed | Tap/click in progress. Color, scale, or opacity change defined. |
| Disabled | When the action isn't available. Clearly distinct from the default state. |

> **Why:** Builders implement what design specifies. If pressed and disabled states aren't designed, they either get skipped or guessed — inconsistently, across every feature, by every builder.

> **Failure mode:** A "disabled" send button rendered identically to an enabled one because no disabled state was ever designed. User taps repeatedly, nothing happens — no feedback, no explanation.

---

### All UI meets WCAG AA contrast — 3:1 for components and icons, 4.5:1 for body text

Every colour token must be verified at the time it's chosen. Document the contrast ratio alongside the token value in code and design.

- **UI components and icons:** 3:1 minimum (WCAG 1.4.11 Non-text Contrast)
- **Body text and UI text:** 4.5:1 minimum (WCAG 1.4.3 Contrast Minimum)

> **Why:** WeekendWare products are built for real people with real conditions. Contrast is a baseline accessibility requirement — not a polish concern — on every project. For Basil specifically: diabetic retinopathy is a common long-term complication of T1D and directly reduces contrast sensitivity.

> **Failure mode:** Night nav inactive icons at `#2E4A35` on `#131E14` produced a 1.8:1 ratio — far below the 3:1 minimum for icons. Fixed to `#6A9470` (~4.9:1) after an accessibility review. Would have shipped invisible to a significant portion of the target user population.

*Source: WCAG 2.1 AA — 1.4.3 Contrast (Minimum) for text, 1.4.11 Non-text Contrast for UI components and icons*

---

### Color is not the only means of conveying interactive state — WCAG 1.4.1

Color alone cannot be the only way to distinguish an interactive state — active vs inactive, selected vs unselected, error vs normal. Every state change that uses color must also use a second differentiator: shape (filled pill, underline), label change, pattern, icon change, or weight change.

> **Why:** WCAG 1.4.1: *"Color is not used as the only visual means of conveying information, indicating an action, prompting a response, or distinguishing a visual element."* Users with colour blindness, low contrast vision, or screen glare cannot reliably distinguish a teal icon from a grey one.

> **Failure mode:** Basil's active nav item was differentiated solely by sage green icon vs grey inactive — a single signal invisible to anyone with reduced colour sensitivity. Fix: MD3 active indicator pill (shape change) in addition to the colour change.

*Source: WCAG 2.2 — 1.4.1 Use of Color*

---

### Minimum touch target — 24×24px CSS (WCAG 2.5.8); 48dp Android, 44pt iOS

Every interactive element must have a minimum tappable area. Visual size can be smaller than the hit area. The hit area is what counts.

| Standard | Minimum | Notes |
|----------|---------|-------|
| WCAG 2.5.8 AA (2022) | 24×24 CSS px | Absolute floor; or offset so a 24px circle doesn't intersect adjacent targets |
| WCAG 2.5.5 AAA | 44×44 CSS px | Stricter; matches Apple and approaches Android |
| Android accessibility | 48×48 dp | Required; M3 built-ins enforce this automatically |
| Apple HIG | 44×44 pt | Required for all interactive elements |

> **Why:** Small tap targets are one of the most common mobile accessibility failures. They affect older users, people with motor impairments, and everyone using their phone with one hand on a bus.

> **Failure mode:** A 24×24dp icon button with no hit area expansion. User taps repeatedly; the target captures maybe 30% of attempts. Under stress — a diabetic emergency, distracted moment — this is a failure of the product's core promise.

*Source: WCAG 2.2 — 2.5.8 Target Size Minimum; Android Accessibility Guidelines; Apple HIG*

---

### All code documented using the platform-appropriate documentation library

Every public class, function, method, property, and interface must have a documentation comment using the correct tool for the language. Use structured tags — don't freeform comment when a tag exists.

| Platform | Tool | Style | Key tags |
|----------|------|-------|----------|
| Kotlin | KDoc (processed by Dokka) | `/** */` | `@param` `@return` `@throws` `@property` `@constructor` `@see` |
| Rust | Rustdoc | `///` for items, `//!` for modules | `# Examples`, `# Panics`, `# Errors` |
| JavaScript / TypeScript | JSDoc / TSDoc | `/** */` | `@param` `@returns` `@throws` `@type` `@template` |
| Swift | DocC | `///` or `/** */` | `- Parameter:` `- Returns:` `- Throws:` |

> **Failure mode:** An undocumented function with a non-obvious contract gets called with the wrong assumptions. Worse: JavaDoc syntax used in a Kotlin file — Dokka parses it differently, IDEs show broken tooltips, and it signals the author didn't know the platform.

*Source: Kotlin KDoc + Dokka docs, Rust Reference (doc comments), TSDoc spec, Apple DocC*

---

## Build & DevOps

Universal build rules. Platform-specific rules (Gradle, Xcode, etc.) live in the platform rules file.

| Rule |
|------|
| CI job order: tests pass first, platform builds run in parallel after |
| PR check = lint + fast tests. Merge to `develop` = full platform matrix. |
| `secrets.sample.properties` committed — actual keys file is not |
| CI never builds `prod` flavor from a feature branch |

### CI job order: tests pass first, platform builds run in parallel after

```yaml
job: test           # ubuntu, fast
job: build-android  # needs: test
job: build-ios      # macos-14, needs: test
```

> **Failure mode:** Expensive iOS macOS runner spins up and times out on code with broken Kotlin logic. Costs money.

*Source: JetBrains official GitHub Actions for KMP docs*

---

### PR check = lint + fast tests. Merge to `develop` = full platform matrix.

Lightweight PR feedback stays fast. Full build matrix on merge prevents platform-breaking changes from reaching `develop`.

---

### `secrets.sample.properties` committed — actual keys file is not

Documents every key name without values. New developers copy it. The actual secrets file is in `.gitignore`.

> **Failure mode:** A developer copies a teammate's real values into their local setup. Dev-tier keys connect to the wrong backend project.

---

### CI never builds `prod` flavor from a feature branch

Only `develop → staging` and `main → prod`. Tag-triggered workflows gate production releases.

> **Failure mode:** A developer pushes a half-finished feature. CI assembles a prod build. It gets accidentally distributed.

---

## QA & Testing

Universal testing rules. Platform-specific testing rules live in the platform rules file.

| Rule |
|------|
| Fakes before mocks — mocking frameworks must not go in shared test sources |
| Every ViewModel state transition has a test — loading, success, error, empty minimum |

### Fakes before mocks

Android docs (March 2026): *"First, use a fake. If not possible, mock. Do not install a mocking framework unless clearly necessary."*

MockK and Mockito are JVM-only — they cannot be added to shared or cross-platform test sources.

> **Failure mode:** A mocking framework added to shared tests. iOS and Desktop test compilation fails. Tests move to `androidTest` only. iOS bugs in shared code go undetected.

*Source: Android Developers, Now in Android (zero mocking libraries)*

---

### Every ViewModel state transition has a test — loading, success, error, empty minimum

If a state has a `@Preview`, it has a test. Error and empty states are exactly the states that break in production.

> **Failure mode:** A Supabase auth expiry causes the error state to emit an `AuthException` instead of a `StringResource`. User sees a crash instead of a recoverable error message.

*Source: Android Developers (what to test), kmpship.app 2025*

---

## Security

> [!CAUTION]
> Rules marked **Blocker** must pass before any real user traffic reaches the app.

| Rule | Level |
|------|-------|
| API keys stay on the server — never in the binary or client config | **Blocker** |
| No credentials, tokens, or user IDs in any log output | **Blocker** |
| Session timeout after inactivity — 15 minutes, configurable | **Blocker** |
| HTTPS enforced on staging and prod — no plaintext traffic | **Blocker** |
| Every request to the API authenticated — no anonymous endpoints | **Blocker** |
| Server-side rate limiting — independent of client UX | **Blocker** |
| No request bodies logged on the server | **Blocker** |
| Security reviews any new third-party SDK before it ships | Ongoing |

### API keys stay on the server — never in the binary or client config `BLOCKER`

The server holds all API keys. The mobile client authenticates with a JWT. The client never sees the underlying service key.

> **Failure mode:** Key extracted from APK via `apktool` → unlimited API spend on attacker's behalf.

---

### No credentials, tokens, or user IDs in any log output `BLOCKER`

Applies to `Log.d()`, `println()`, `tracing::info!()`, and any other logging call. The same rule applies to every platform and every layer of the stack.

*Source: OWASP Mobile M1 2024/2025*

---

### Session timeout after inactivity — 15 minutes, configurable `BLOCKER`

HIPAA guidance: automatic session termination after inactivity. Applies to any authenticated app — not just health products.

*Source: OWASP Mobile M3, HIPAA guidance*

---

### HTTPS enforced on staging and prod — no plaintext traffic `BLOCKER`

OWASP: certificate pinning is *not recommended* in 2025 — rely on default PKI trust stores + HSTS. But TLS itself is non-negotiable on staging and prod.

*Source: OWASP Mobile M5*

---

### Every request to the API authenticated — no anonymous endpoints `BLOCKER`

An unauthenticated endpoint proxying to an AI API is a direct cost vector.

> **Failure mode:** Someone discovers the endpoint. Unlimited API requests, zero auth friction. Bill grows until the app goes offline.

---

### Server-side rate limiting — independent of client UX `BLOCKER`

Client-side rate limiting (disabling a button) is UX, not security. The API enforces per-user limits regardless of what the client does.

---

### No request bodies logged on the server `BLOCKER`

Conversation content and user data must never land in server logs. `tracing::info!("{:?}", request_body)` on the server is just as dangerous as logging on the client.

---

### Security reviews any new third-party SDK before it ships

Analytics, A/B testing, and monetisation SDKs must be audited for data access or transmission before being added. This is how sensitive user data ends up in ad networks.

*Source: OWASP Mobile M6*
