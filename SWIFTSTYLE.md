# SWIFTSTYLE.md

Authoritative Swift code style and project layout conventions. All code must conform — modify the code to match this file, not the reverse. `Foo`/`Bar`/`Baz` are placeholders; `App` and `DebugLog` are real conventions.

## A. Critical Constraints

### A.1 Forbidden

- A.1.a. Never define a type with multiple extensions of itself. Sole exception: protocol default implementations in a `protocol` extension.
- A.1.b. Never write an extension on a type you don't own; wrap and forward instead.
- A.1.c. Never nest types — no `struct`/`class`/`enum`/`actor` declared inside another.
- A.1.d. Never use `typealias` unless the language leaves no alternative.
- A.1.e. Never use `DispatchQueue`; use structured concurrency (async/await, actors).
- A.1.f. Never use class inheritance except to subclass a framework type we don't own, like `UIViewController`.

### A.2 Formatting & Style

- A.2.a. **Annotate variable declarations with explicit types** — don't rely on inference.

```swift
let foo: FooModel = FooModel(name: name, tags: [], category: "General")   // Good
let foo = FooModel(name: name, tags: [], category: "General")             // Bad
```

- A.2.b. **Keep calls, initializers, and signatures on one line** — don't break parameters across lines. Exception: SwiftUI view-modifier chains stay multi-line.

```swift
func submit(_ item: String, using bar: BarModel) async -> (response: String, count: Int?)? {   // Good
func submit(                                                                                    // Bad
    _ item: String,
    using bar: BarModel
) async -> (response: String, count: Int?)? {
```

### A.3 Warnings, Linting & Strict Modes

- A.3.a. Ship warning-free; zero SwiftLint violations. Both block "done".
- A.3.b. Fix warnings at the source.
- A.3.c. Allow a warning only at the narrowest scope (`#warning`, `@available`-guarded deprecation, documented pragma), with an inline comment saying why.
- A.3.d. Never disable a SwiftLint rule — no `// swiftlint:disable`, `disabled_rules`, or config loosening. Fix the code, not the rule.
- A.3.e. A genuine rule misfit changes the shared `swiftlint.yml` centrally, never a local override.

## B. Architecture & File Organization

### B.1 Project Layout

- B.1.a. **Separate UI from logic structurally.**
  - B.1.a.i. Views, modifiers, view-only helpers → `Views/`.
  - B.1.a.ii. Models, services → feature folders.
  - B.1.a.iii. A view holding parsing, transformation, or persistence is a defect.
  - B.1.a.iv. A model/service never imports SwiftUI or references view types.
- B.1.b. **Group by feature, not by layer.**
  - B.1.b.i. A feature's `*Model.swift`, `*Service.swift`, and tests share one camelCase folder.
  - B.1.b.ii. Views → `Views/`. App-level and shared code (`@main`, `AppService`, `DebugLog`) → `app/` (or `shared/`).
  - B.1.b.iii. File names match the primary type.

```text
AppName/
  App/     AppService.swift, DebugLog.swift   # app-level + shared
  Foo/     FooModel.swift, FooService.swift, FooServiceTests.swift
  Bar/     BarModel.swift, BarService.swift
  Views/   AppView.swift, ...                 # UI, not feature-grouped
AppNameUITests/  AppNameUITests.swift
```

- B.1.c. **Folders are folders — never path components in filenames.**
  - B.1.c.i. A subfolder is a real folder on disk and a nested `PBXGroup`.
  - B.1.c.ii. Each `PBXFileReference` carries a bare `path`; its `PBXGroup` `path` is the folder.
  - B.1.c.iii. Baking the folder into the filename defeats the structure.

```text
foo (PBXGroup, path = foo)                          // Good — nested group, bare filename
  └─ FooService.swift (path = FooService.swift)
bar (PBXGroup, path = bar)                          // Bad — folder baked into filename
  └─ foo/FooService.swift (path = foo/FooService.swift)
```

- B.1.d. **Swift Package modules follow SPM layout** — the only deviation from the flat layout.
  - B.1.d.i. Sources in `Sources/<Module>/`, tests in `Tests/<Module>Tests/`; feature grouping preserved within.
  - B.1.d.ii. One library product; package `name`, product, and target share one prefixed name (`FooPro`); root folder matches.
  - B.1.d.iii. Each manifest repeats build settings (`platforms`, `swiftLanguageModes`, `swiftSettings`) — SPM has no shared-include.
  - B.1.d.iv. The app injects runtime resources; the package keeps a minimal in-code fallback. Test fixtures under `Tests/`.

### B.2 File-Level Layout

- B.2.a. One primary `struct`/`class` per file; small non-shared supporting types (enums, helpers) may share it.
- B.2.b. Error enums at the top, before the primary type.
- B.2.c. File name matches the primary type.

### B.3 Access Control & Visibility

- B.3.a. Default to internal. Expose `public` only where needed for clean interfaces or testability.

### B.4 Dependencies & Standard Library

- B.4.a. Apple frameworks and the standard library only — no external dependencies.
- B.4.b. Import order: `Foundation`, then Apple frameworks, then project modules.

## C. Code Construction & Logic

### C.1 Logic Layout & Readability

- C.1.a. **Three layers — roles, not directories.**
  - C.1.a.i. **Models** — `Codable` value-semantics structs that carry state.
  - C.1.a.ii. **Services** — observable reference-type classes holding logic and state, operating on models.
  - C.1.a.iii. **Views** — SwiftUI views observing services and rendering UI.
  - C.1.a.iv. Data flows up (services → views) via observation; intent flows down via method calls.
  - C.1.a.v. A view never reaches around a service for its own logic; a service never knows its observing views.
- C.1.b. **State observation.**
  - C.1.b.i. Use `@Observable` — replaces `ObservableObject`/`@Published`, no `import Combine`.
  - C.1.b.ii. A view owns the object with `@State`, derives bindings with `@Bindable`.
  - C.1.b.iii. Mark UI-driving types `@MainActor`.
  - C.1.b.iv. Fall back to `ObservableObject` + `@Published` only pre-Observation (iOS 17 / macOS 14).
- C.1.c. **Logic extraction.**
  - C.1.c.i. Extract logic into focused service files — stateless `enum` types with `static` methods, callable in tests without instantiation.
  - C.1.c.ii. Parsing → `FooParseService.swift`, not `FooModel.swift`. Value-mapping → `BarMapService.swift`, not a view.
  - C.1.c.iii. Models stay pure data; views stay thin.
- C.1.d. **Prefer encapsulation over extensions** — keep behavior in the type's own body.
- C.1.e. **Prefer computed properties over stored flags when the value derives from other stored properties** — avoids stale state and `Codable` migration issues.

```swift
public var isPublic: Bool { username.isEmpty && password.isEmpty }   // Good — derived
public var isPublic: Bool                                            // Bad — can desync
```

### C.2 Naming Conventions

| Element                           | Convention                     | Example                   |
| --------------------------------- | ------------------------------ | ------------------------- |
| Type (struct/class/enum/protocol) | UpperCamelCase                 | `FooModel`, `BarService`  |
| Function / method                 | lowerCamelCase, verb-based     | `fetchItems()`            |
| Variable / property / constant    | lowerCamelCase                 | `fooURL`, `maxRetryCount` |
| Enum case                         | lowerCamelCase                 | `case foo`                |
| Boolean                           | `is`/`has`/`should` prefix     | `isLoadingFoo`            |
| Protocol                          | `Protocol` suffix              | `ServiceProtocol`         |
| Model                             | `Model` suffix, never `Result` | `FooModel`                |
| Acronym                           | all caps                       | `URL`, `API`, `UUID`      |

- C.2.a. Give a family of related types a consistent semantic prefix naming its domain (`FooModel`/`FooService`/`FooView`); app-level types use `App`.

### C.3 Control Flow & Branching

- C.3.a. Prefer `guard` and early returns over nested `if-let` for preconditions — keeps the happy path unindented.

### C.4 Error Handling & Guard Clauses

- C.4.a. Prefer `throws` over `Result` or optionals; one custom error enum per domain.
- C.4.b. If a call can't fail worth reacting to, prefer a non-throwing API over swallowing a `throws` call.
- C.4.c. Never silently discard an error — surface a user-visible signal (alert, error state, disabled action), an `os_log.debug()` diagnostic, or deliberate recovery.
- C.4.d. Never swallow via `try?`, an empty `catch {}`, or an ignored failure-encoding `Result`/optional. Exception: test code, where an unexpected error must fail the test.

```swift
_ = try? FooService.create(value, in: context)     // Bad — failure vanishes

do { try FooService.create(value, in: context) }    // Good — surface to user
catch { self.saveError = error }

do { try FooService.create(value, in: context) }    // Good — log & continue deliberately
catch { log.debug("foo create failed: \(error)") }
```

- C.4.e. **Never force-unwrap, force-try, or force-cast** — `!`, `try!`, `as!` crash on `nil`/error/mismatch and discard the diagnostic. Anywhere, including tests.
  - C.4.e.i. Optionals: `guard let`/`if let`, or `??` default.
  - C.4.e.ii. Throwing calls: `try` in a `throws` context, or `do`/`catch`.
  - C.4.e.iii. Casts: conditional `as?`, handle `nil`.
  - C.4.e.iv. Tests: `try` in a throwing `@Test`, `#expect(throws:)`, `try #require(...)` to unwrap.
  - C.4.e.v. Truly-unreachable: `do`/`catch` calling `fatalError("…")` — the message is the diagnostic a force operator throws away.

```swift
let first: Item = items.first!                       // Bad — crashes, no context
let first: Item = try #require(items.first)          // Good (test)
do { container = try ModelContainer(for: schema) }    // Good — trap with message
catch { fatalError("Failed to build the store: \(error)") }
```

## D. Systems & State Management

### D.1 Concurrency & Threading

- D.1.a. Prefer async/await over callbacks and completion handlers.
- D.1.b. Prefer `Sendable` types over actors.

### D.3 Logging, Diagnostics & Telemetry

- D.3.a. **Use `os.Logger`, never `print()` with the sole exception of CLI mains, in model and service code** — debug messages are excluded from release output by default, no `#if DEBUG` needed.

```swift
import os

let log = Logger(subsystem: "com.example.app", category: "FooService")
log.debug("parsed \(count) items")   // debug level: omitted from release output by default
```

## E. Quality, Tooling & Frontend

### E.1 Documentation & Comments

- E.1.a. **Give every method and property a `///` doc comment.** `///` for doc comments on declarations only; `//` for inline comments in bodies.

```swift
/// Fetch a single foo by ID
public static func foo(_ db: OpaquePointer, id: String) -> FooModel? {
    // Query the joined view   <- inline uses //, not ///
}
```

### E.2 UI Patterns

- E.2.a. Never mutate `@State` during `body` evaluation — read in `body`, mutate in action closures, `.onAppear`, or event handlers. Else "Modifying state during view update".
- E.2.b. Never hardcode fixed widths or sizes — use spacing, let content size itself; fixed dimensions break across platforms and accessibility.
- E.2.c. Prefer inclusive semantics — `languages` (selected), not `excludedLanguages`.
- E.2.d. Prefer reusable views/modifiers over inline redeclaration — extract shared patterns into named types.
- E.2.e. **Every `Button` carries a tooltip and an accessibility identifier.**
  - E.2.e.i. `.help("…")` — concise verb-first tooltip (`"Resume a saved session"`); hover affordance + baseline accessibility label.
  - E.2.e.ii. `.accessibilityIdentifier("…")` — stable, test-facing, never localized. Dotted `lowerCamelCase` scoped by owning view, `<viewScope>.<action>`, naming what the button does, not its label (`projectWindow.resumeSession`, `launcher.openFolder`).
  - E.2.e.iii. `Button { } label: { … }` form: attach both modifiers to the `Button`, not the label.
  - E.2.e.iv. System-container buttons (`.alert`, `.confirmationDialog`) show no `.help` — identifier only.

### E.3 Build, Tooling & Task Runners

- E.3.a. All build settings in `.xcconfig`, referenced per target via `baseConfigurationReference`. Never inline `buildSettings` in the project file.
- E.3.b. Keeps the pbxproj minimal and settings diff-reviewable.

### E.4 Testing Standards & Test Data

- E.4.a. Test files next to the source they test (`FooService.swift` → `FooServiceTests.swift`, same folder). Never a separate `tests/` directory.
- E.4.b. Prefer fixture data over mocks — the models/services pattern maximizes testability without mocking.
