# iOS Rules

Rules for any iOS target in a WeekendWare project. Read [`weekendware-rules.md`](weekendware-rules.md) first — universal rules apply here too. For CMP projects also read [`kmp-rules.md`](kmp-rules.md). These rules extend, not replace, those files.

---

## Project Structure

| Rule |
|------|
| `iosApp/` is the Xcode project root — never move it |
| Build settings live in `.xcconfig` files — never in `project.pbxproj` |
| One `xcconfig` file per environment: `Config.xcconfig` (shared), `Dev/Staging/Prod.xcconfig` (per-env overrides) |
| `TEAM_ID` in `Config.xcconfig` must be left blank in source control |
| Only the Sentry XCFramework is vendored — all other dependencies via SPM |

### Build settings live in `.xcconfig` files

`project.pbxproj` is a binary-ish format that produces unusable diffs on merge conflicts. All per-environment settings (`BUNDLE_ID`, `API_URL`, encryption flags) live in `.xcconfig` files, which are plain text and diff cleanly.

> **Failure mode:** A `PRODUCT_BUNDLE_IDENTIFIER` overridden directly in the project file silently defeats the xcconfig. The dev flavor ships with the prod bundle ID.

*Source: Apple Technical Q&A QA1864 — Setting build variables with xcconfig files; iOS Dev community consensus (donnywals.com, pointfree.co)*

---

## CMP Entry Point

| Rule |
|------|
| `ComposeUIViewController` is the only way to host Compose content on iOS |
| The Swift wrapper always sets `.ignoresSafeArea(.keyboard)` |
| `initKoin()` and `initSentry()` are called inside `MainViewController()` before `ComposeUIViewController` |
| Guard `initKoin()` against double-initialisation if the view controller can be called in SwiftUI previews |

### `ComposeUIViewController` is the only host for Compose on iOS

Never use a `UIHostingController` wrapping SwiftUI to host CMP content, and never attempt to embed CMP composables inside a SwiftUI `View` hierarchy directly. The only supported bridge is `UIViewControllerRepresentable` wrapping a `ComposeUIViewController`.

```swift
// ContentView.swift — correct pattern
struct ComposeView: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> UIViewController {
        MainViewControllerKt.MainViewController()
    }
    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {}
}

struct ContentView: View {
    var body: some View {
        ComposeView()
            .ignoresSafeArea(.keyboard) // Compose manages its own keyboard avoidance
    }
}
```

> **Why:** Without `.ignoresSafeArea(.keyboard)`, UIKit raises the entire view when the keyboard appears — Compose then applies its own inset on top, resulting in double-offset layouts. `ComposeUIViewController` is the only officially supported host; anything else is unsupported and breaks CMP's gesture handling and focus management.

*Source: JetBrains CMP iOS integration docs — "Add Compose Multiplatform to an existing iOS app" (jetbrains.com/help/kotlin-multiplatform-dev); JetBrains sample app iosApp/ContentView.swift*

---

### Guard `initKoin()` against double-initialisation

`MainViewController()` initialises Koin and Sentry. If SwiftUI ever recreates the view controller (e.g. in previews or via state changes), a second call to `initKoin()` throws a `KoinAlreadyStartedException` and crashes.

```kotlin
import org.koin.core.error.KoinApplicationAlreadyStartedException

fun MainViewController() = run {
    initSentry()
    try {
        initKoin()
    } catch (_: KoinApplicationAlreadyStartedException) {
        // Safe: SwiftUI previews may reconstruct this controller more than once.
    }
    ComposeUIViewController { App() }
}
```

Koin 4.x changed the exception name from `KoinAlreadyStartedException` to `KoinApplicationAlreadyStartedException`. In Koin 4.x on Kotlin/Native, `GlobalContext` is not a directly importable singleton — the native implementation uses `globalContextByMemoryModel()` internally. The try-catch approach works across all Koin versions and all targets.

> **Failure mode:** SwiftUI previews instantiate the root controller on every preview render. Without the guard, the preview crashes with `KoinApplicationAlreadyStartedException` — no crash report, just a blank preview.

*Source: Koin documentation — error package (insert-koin.io/docs); Koin 4.x source — GlobalContext.kt nativeMain (github.com/InsertKoinIO/koin); Philipp Lackner KMP courses (2025–26)*

---

## Kotlin/Native Cinterop (K2)

These rules apply to any `iosMain` Kotlin code that calls into Apple frameworks via cinterop. K2 (Kotlin 2.x) made significant breaking changes to the cinterop API surface. All rules in this section were verified against Kotlin 2.1.21.

| Rule |
|------|
| Never use `import kotlinx.cinterop.*` (wildcard) |
| `NSData.getBytes(_:length:)` is not exposed in K2 — use `base64EncodedStringWithOptions` + Kotlin `Base64` |
| `NSData.bytes` (NS_RETURNS_INNER_POINTER) is not exposed in K2 |
| `NSString(data:encoding:)` is not a valid Kotlin constructor in K2 — only `NSString()` and `NSString(coder:)` exist |
| ARC-managed objects (NSData, NSString from Keychain) must NOT be passed to `CFRelease` |
| `ObjCObjectVar<*>` is the K2 replacement for the removed `CFTypeRefVar` |
| `alloc<ObjCObjectVar<*>>()` requires `@OptIn(BetaInteropApi::class)` |
| Always call `CFRelease` on CF objects you own (CFStringCreateWithCString, CFDataCreate, CFDictionaryCreateMutable) |
| `usePinned { }` is the only safe way to pass a Kotlin ByteArray pointer into a C/CF API |

### Never use `import kotlinx.cinterop.*` (wildcard)

The wildcard import brings `fun <T : CVariable> CValues<T>.getBytes(): ByteArray` into scope. This extension shadows any ObjC method named `getBytes` on any type — including `NSData.getBytes(_:length:)`. The compiler resolves to the wrong overload and the ObjC method becomes unreachable.

Always import specifically:
```kotlin
import kotlinx.cinterop.ExperimentalForeignApi
import kotlinx.cinterop.ObjCObjectVar
import kotlinx.cinterop.addressOf
import kotlinx.cinterop.alloc
import kotlinx.cinterop.memScoped
import kotlinx.cinterop.ptr
import kotlinx.cinterop.reinterpret
import kotlinx.cinterop.usePinned
import kotlinx.cinterop.value
```

> **Why:** `kotlinx.cinterop.*` is designed for C struct interop. Mixing C and ObjC interop in the same file with a wildcard import causes name collisions that produce misleading error messages — the real problem (wrong import scope) is invisible.

*Source: JetBrains Kotlin/Native C interop docs (kotlinlang.org/docs/native-c-interop.html); JetBrains Kotlin/Native ObjC/Swift interop docs (kotlinlang.org/docs/native-objc-interop.html)*

---

### `NSData` → `ByteArray` in Kotlin/Native 2.x K2

Neither `NSData.bytes` (excluded due to `NS_RETURNS_INNER_POINTER`) nor `NSData.getBytes(_:length:)` (shadowed by cinterop wildcard, or absent from K2 bindings) is reliably accessible as a direct Kotlin call.

**Working pattern:**
```kotlin
import kotlin.io.encoding.Base64
import kotlin.io.encoding.ExperimentalEncodingApi

@OptIn(ExperimentalEncodingApi::class)
val keyString = data.base64EncodedStringWithOptions(0UL)
    ?.let { Base64.decode(it).decodeToString() }
```

`NSData.base64EncodedStringWithOptions(_:)` returns a Kotlin `String` via the NSString bridge and is reliably available in K2. `kotlin.io.encoding.Base64` (`@ExperimentalEncodingApi`) is available in Kotlin 2.x.

> **Why:** The round-trip is semantically correct: `CFDataCreate` copies raw bytes in; `base64EncodedStringWithOptions` encodes them; `Base64.decode` recovers the original bytes; `decodeToString()` converts to string. No pointer arithmetic.

*Source: Kotlin 2.1 What's New (kotlinlang.org/docs/whatsnew21.html); Kotlin/Native interop evolution discussion — YouTrack KT-66452; Kotlin stdlib `kotlin.io.encoding.Base64` API reference*

---

### ByteArray → CFData (write path)

`CFDataCreate` is the reliable way to create a `CFDataRef` from a Kotlin `ByteArray`:

```kotlin
val cfData = keyBytes.usePinned { pinned ->
    CFDataCreate(null, pinned.addressOf(0).reinterpret(), keyBytes.size.toLong())!!
}
// ... use cfData ...
CFRelease(cfData)  // CFDataCreate returns a +1 retained object
```

`usePinned` pins the `ByteArray` in GC memory for the duration of the lambda. `CFDataCreate` copies the bytes internally, so `cfData` outlives the `usePinned` scope safely.

*Source: Apple Core Foundation — CFDataCreate reference (developer.apple.com/documentation/corefoundation/cfdatacreate(_:_:_:)); JetBrains Kotlin/Native C interop — Pinned objects (kotlinlang.org/docs/native-c-interop.html#pass-pointers-to-bindings)*

---

### ARC vs CF ownership

| Object origin | Ownership | Action |
|--------------|-----------|--------|
| `CFStringCreateWithCString(...)` | CF, you own it (+1) | `CFRelease(ref)` |
| `CFDataCreate(...)` | CF, you own it (+1) | `CFRelease(ref)` |
| `CFDictionaryCreateMutable(...)` | CF, you own it (+1) | `CFRelease(ref)` |
| `SecItemCopyMatching` result via `ObjCObjectVar<*>` | ARC-managed NSData | Do NOT `CFRelease` |
| Any `NSString`, `NSData` from ObjC methods | ARC-managed | Do NOT `CFRelease` |

> **Failure mode:** `CFRelease` on an ARC-managed object over-releases it. ARC releases it again at scope exit — double-free crash. Silent in debug mode on some Xcode versions; crashes in TestFlight builds.

*Source: Apple Memory Management Programming Guide for Core Foundation (developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt); Kotlin/Native ObjC interop — memory management (kotlinlang.org/docs/native-objc-interop.html#memory-management)*

---

## iOS Security

| Rule | Level |
|------|-------|
| Database encryption key generated with `SecRandomCopyBytes` — never `Random.Default` | **Blocker** |
| Keychain items use `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` | **Blocker** |
| Database encryption key stored per-device in Keychain — never hardcoded | **Blocker** |
| `OTHER_LDFLAGS` in `Config.xcconfig` must link SQLCipher before production | **Blocker** |
| File Data Protection class: `NSFileProtectionCompleteUnlessOpen` minimum for app data | Required |
| App Transport Security: no `NSAllowsArbitraryLoads` in production `Info.plist` | **Blocker** |

### Use `SecRandomCopyBytes` for cryptographic key generation

`kotlin.random.Random.Default` on Kotlin/Native is `XorWowRandom` — a deterministic PRNG, not a CSPRNG. Using it to generate a database encryption key means the key is predictable given a known seed.

```kotlin
val keyBytes = ByteArray(32)
keyBytes.usePinned { pinned ->
    val result = SecRandomCopyBytes(kSecRandomDefault, 32UL, pinned.addressOf(0))
    check(result == 0) { "SecRandomCopyBytes failed: $result" }
}
```

`SecRandomCopyBytes` with `kSecRandomDefault` reads from `/dev/random` on iOS — Apple's hardware-seeded CSPRNG.

> **Failure mode:** A key generated from `Random.Default` with a known or guessable seed is brute-forceable. Database encryption with a weak key provides no meaningful protection.

*Source: Apple Security framework — SecRandomCopyBytes (developer.apple.com/documentation/security/secrandomcopybytes(_:_:_:)); OWASP Mobile Security Testing Guide — MSTG-CRYPTO-6 (mas.owasp.org/MASTG); OWASP Mobile Top 10 — M9: Insecure Data Storage*

---

### Keychain accessibility must be `AfterFirstUnlock` for database keys

`kSecAttrAccessibleWhenUnlocked` blocks access to the database when the app is backgrounded. `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` allows background access while preventing iCloud Keychain sync.

```kotlin
CFDictionarySetValue(attrs, kSecAttrAccessible, kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly)
```

`ThisDeviceOnly` prevents the encryption key from migrating to another device via iCloud Backup — the database on the other device would be unreadable, which is the correct security posture.

> **Failure mode:** `kSecAttrAccessibleWhenUnlocked` causes `errSecInteractionNotAllowed` when the app's background task tries to open the database while the device is locked. `kSecAttrAccessible` without `ThisDeviceOnly` syncs the key to iCloud, making the database decryptable from any device on the account.

*Source: Apple Keychain Services — Item Accessibility Constants (developer.apple.com/documentation/security/keychain_services/keychain_items/item_attribute_keys_and_values); OWASP Mobile Top 10 — M9: Insecure Data Storage; Apple Human Interface Guidelines — Privacy*

---

### SQLCipher must be linked in Xcode before production

The Kotlin `DatabaseDriverFactory` calls the SQLCipher API via the native driver's `encryptionConfig`. Until the SQLCipher SPM package is added in Xcode and `OTHER_LDFLAGS` updated, the system `sqlite3` is used — the key is silently ignored and the database is unencrypted.

Required change in `Config.xcconfig`:
```
OTHER_LDFLAGS = $(inherited) -lSQLCipher
```

> **Failure mode:** The app ships to production with an unencrypted SQLite file. All user conversation history — diabetes management data — is readable from any iTunes backup.

*Source: SQLCipher iOS integration (zetetic.net/sqlcipher/ios-tutorial); OWASP Mobile Top 10 — M9: Insecure Data Storage; KmpSqlencrypt GitHub (github.com/natanfudge/KmpSqlencrypt)*

---

## iOS Lifecycle and Background Behaviour

| Rule |
|------|
| `viewModelScope` coroutines that drive time-sensitive UI (e.g. theme hour) must restart correctly after app foregrounding |
| CMP's `LifecycleOwner` / `AppLifecycleEventObserver` (CMP 1.7+) is the correct hook for lifecycle events — not UIKit lifecycle methods |

### Time-sensitive ViewModels must be pre-warmed

A `viewModelScope` coroutine that polls the current hour (e.g. `BasilThemeViewModel`) continues running in the background on iOS when the app suspends. When the app foregrounds, the `StateFlow` already holds the correct value — no one-frame lag.

This is the correct pattern. The alternative (initialising from `LaunchedEffect` on foreground) produces a single-frame flash of the wrong theme.

*Source: JetBrains CMP lifecycle docs — LifecycleOwner and coroutines in CMP (jetbrains.com/help/kotlin-multiplatform-dev); CMP 1.7 release notes (blog.jetbrains.com)*

---

### Use CMP lifecycle APIs, not UIKit lifecycle methods

Overriding `viewWillAppear` / `viewDidDisappear` on the `UIViewController` returned by `ComposeUIViewController` is not supported. Any override is invisible to Compose and has no effect on the CMP component tree.

Use `LocalLifecycleOwner` / `LifecycleEventEffect` from CMP instead:

```kotlin
val lifecycle = LocalLifecycleOwner.current.lifecycle
LifecycleEventEffect(Lifecycle.Event.ON_RESUME) {
    // runs on foreground
}
```

*Source: JetBrains CMP lifecycle docs (jetbrains.com/help/kotlin-multiplatform-dev/compose-lifecycle.html); Philipp Lackner KMP courses (2025–26)*

---

## Build & CI

| Rule |
|------|
| Pin `macos-14` runner and an `XCODE_VERSION` repo variable in every CI workflow |
| Framework build target for development: `linkDebugFrameworkIosSimulatorArm64` |
| Framework build target for release: `linkReleaseFrameworkIosArm64` |
| Verify `linkDebugFrameworkIosSimulatorArm64` passes in CI before opening any iOS PR |
| Never add CocoaPods — use Swift Package Manager only |
| `Sentry.xcframework` is vendored locally — download link in `README.md` |
| `androidx.lifecycle` must stay at the version compiled with the project's Kotlin version — check ABI compatibility before bumping |

### Pin macOS runner and Xcode version

GitHub updates macOS images on their schedule. An Xcode major version bump silently changes compiler behaviour for Swift and breaks Kotlin/Native cinterop headers.

```yaml
runs-on: macos-14
env:
  XCODE_VERSION: ${{ vars.XCODE_VERSION }}
steps:
  - uses: maxim-lobanov/setup-xcode@v1
    with:
      xcode-version: ${{ env.XCODE_VERSION }}
```

*Source: kmpship.app CI/CD guide; JetBrains KMP GitHub Actions documentation (kotlinlang.org/docs/multiplatform-publish-apps.html)*

---

### `androidx.lifecycle` ABI compatibility

`androidx.lifecycle` 2.10.0 was compiled with Kotlin 2.2.20 (ABI 2.2.0). Basil uses Kotlin 2.1.21 (max ABI 1.201.0). Any `androidx.lifecycle` version compiled with a newer Kotlin than the project's Kotlin version will fail the iOS framework link with an ABI incompatibility error.

Before bumping `androidx.lifecycle`, check the release notes for the Kotlin version it was compiled with. Downgrade if there is a mismatch.

Known safe: `2.9.0` (compiled with Kotlin 2.1.x, compatible with Kotlin 2.1.21).

*Source: Kotlin ABI stability documentation (kotlinlang.org/docs/kotlin-evolution-principles.html); AndroidX lifecycle release notes (developer.android.com/jetpack/androidx/releases/lifecycle)*

---

## QA & Testing

| Rule |
|------|
| iOS framework build (`linkDebugFrameworkIosSimulatorArm64`) is the first iOS gate — must pass before any iOS testing in Xcode |
| `iosMain` code must be reviewed for cinterop correctness before merge — type mismatches and ObjC method resolution failures only appear at compile time, not in unit tests |
| No mocking frameworks in iOS-specific tests — fakes only |
| Test Keychain operations in the simulator — the Keychain is fully functional in `iphonesimulator` target builds |

### iOS framework build is the first gate

`./gradlew linkDebugFrameworkIosSimulatorArm64` compiles all of `commonMain` and `iosMain` against the iOS SDK. This is the only build step that catches:
- Kotlin/Native K2 cinterop regressions
- `expect`/`actual` mismatches on the iOS target
- ABI incompatibilities from library version bumps

It must pass before any Xcode build, device test, or PR review begins.

*Source: JetBrains KMP multiplatform build docs (kotlinlang.org/docs/multiplatform-run-tests.html); kmpship.app CI/CD guide*

---

### No mocking frameworks in iOS-specific tests

Kotlin/Native does not support runtime bytecode manipulation. Most JVM mocking libraries (Mockito, MockK) do not work in `iosTest`. Use hand-written fakes in a `fake/` or `test-fixtures/` package.

> **Failure mode:** MockK is added to `commonTest`. Passes on `desktopTest`. Crashes at link time on `iosTest` with a linker error or `NoSuchMethodException` at runtime.

*Source: MockK documentation — platform support matrix (mockk.io); Philipp Lackner KMP testing patterns (2025–26)*
