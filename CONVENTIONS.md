# CONVENTIONS.md

Authoritative conventions for writing style guides in this repo. All `<LANG>STYLE.md` files must conform. This file is itself written under the Catalog it defines (§B.2) — the same five groups every guide uses.

## A. Critical Constraints

### A.1 Forbidden

- A.1.a. Never write narrative, motivation, or history. State the rule, not the story behind it.
- A.1.b. Never hedge — no "consider", "prefer when possible", "it's recommended", "generally". A rule is mandatory or it is not a rule.
- A.1.c. Never state a rule without a one-line rationale and an example.
- A.1.d. Never soften a rule to match existing code. The guide is authoritative; conflicting code is wrong.
- A.1.e. Never leave a Swift rule unmirrored in `swiftlint.yml` / `swiftformat.cfg`, or vice versa.
- A.1.f. Never add a guide without adding its bullet to `README.md`.

### A.2 Formatting & Style

- A.2.a. A rule is: imperative statement, one-line rationale, paired Good/Bad example.
- A.2.b. Default to bullet fragments — a rule is not a sentence. Drop articles and filler.
- A.2.c. Use a bold-lead item only when a code example or sub-items follow it.
- A.2.d. Write rules as commands: "do X", "never Y", "return T". Drop the subject.

```markdown
Prefer guard and early returns over nested if-let for preconditions. <- Good
You should try to use guard statements where it makes sense to do so. <- Bad — hedged, wordy
```

- A.2.e. Every rule carries a minimal example — smallest snippet that shows it. On accept/reject, mark both with inline `// Good` / `// Bad` on the divergent line. Use `Foo`/`Bar` placeholders; flag real conventions in the preamble.

```swift
let foo: FooModel = FooModel(name: name)   // Good — explicit type
let foo = FooModel(name: name)             // Bad  — relies on inference
```

## B. Architecture & File Organization

### B.1 Project Layout

- B.1.a. Style guide: `<LANG>STYLE.md`, uppercase (`GOSTYLE.md`, `SWIFTSTYLE.md`). Migration notes: `<LANG>MIGRATE.md`. Config files stay lowercase (`swiftlint.yml`).
- B.1.b. Register each guide with a one-line `README.md` bullet — `[FILE.md](FILE.md) — one-line scope.` Nothing generates the index.

### B.2 File-Level Layout

- B.2.a. Open with an H1 matching the filename, then a one-to-two-line preamble stating scope and authority. No table of contents, no intro section.
- B.2.b. Groups are `##`, rules are `###`, items are bullets. Rules sit under the five canonical groups:

```text
A. Critical Constraints          — fail-fast, highest priority
   A.1 Forbidden · A.2 Formatting & Style · A.3 Warnings, Linting & Strict Modes
B. Architecture & File Organization
   B.1 Project Layout · B.2 File-Level Layout · B.3 Access Control & Visibility · B.4 Dependencies & Standard Library
C. Code Construction & Logic
   C.1 Logic Layout & Readability · C.2 Naming Conventions · C.3 Control Flow & Branching · C.4 Error Handling & Guard Clauses
D. Systems & State Management
   D.1 Concurrency & Threading · D.2 Resource Management · D.3 Logging, Diagnostics & Telemetry
E. Quality, Tooling & Frontend
   E.1 Documentation & Comments · E.2 UI Patterns · E.3 Build, Tooling & Task Runners · E.4 Testing Standards & Test Data
```

- B.2.c. Group letters and rule numbers are canonical — identical across all guides. Include only applicable rules; an inapplicable rule is a gap (skip the number, never reuse it). Language-specific detail lives only in the lettered items under a rule.
- B.2.d. IDs are dotted: group letter · rule number · item letter (`A.1.a`); sub-items append lowercase roman (`A.1.a.i`).
- B.2.e. Cite as `<GUIDE> <id>` — `SWIFT A.1.a`, `SWIFT C.4.e.iii`. `<GUIDE>` is one of `SWIFT`, `GO`, `ZIG`, `SQL`. Table rows cite by row label (`SWIFT C.2 Boolean`).
- B.2.f. Group and rule labels are stable across guides; item and sub-item letters are positional within a rule and relabel on reorder.

## C. Code Construction & Logic

### C.2 Naming Conventions

- C.2.a. `C.2 Naming Conventions` in a guide carries a table mapping each element kind to its convention and an example. One row per kind.

```markdown
| Element | Convention | Example              |
| ------- | ---------- | -------------------- |
| Struct  | PascalCase | `Config`, `DBClient` |
| Field   | snake_case | `max_retries`        |
```

## E. Quality, Tooling & Frontend

### E.3 Build, Tooling & Task Runners

- E.3.a. A guide with an enforcing config (SwiftLint, SwiftFormat) is one standard in two forms — every rule change edits both.
- E.3.b. Every config entry that looks like a mistake carries an inline comment explaining why.
