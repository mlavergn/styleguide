# Code Style

Code style guides for various coding languages.

## Usage

Clone this repo beside the projects that use it, so each project reaches it as `../styleguide`. A project's Makefile starts with:

```make
# Optional include to extend the makefile targets
-include ../styleguide/Makefile.in
```

That supplies the canonical `make format`, `make lint`, `make validate`, and `make buildxpin`, which pins zigbuildx in `build.zig.zon` to its newest release. The project defines `clean`, `build`, and `test`, and sets `ZIG_DIRS`, `LINT_EXCLUDE`, or extra `validate:` prerequisites after the include when it needs to.

## Style Guides

- [CONVENTIONS.md](CONVENTIONS.md) — Conventions for writing the style guides themselves (terse, imperative, rule-per-section).
- [GOSTYLE.md](GOSTYLE.md) — Go code style and project layout conventions.
- [SWIFTSTYLE.md](SWIFTSTYLE.md) — Swift code style and project layout conventions, with a strict UI/logic separation.
- [ZIGSTYLE.md](ZIGSTYLE.md) — Zig code style and project layout conventions, emphasizing avoidance of deprecated APIs.
- [ZIGMIGRATE.md](ZIGMIGRATE.md) — Zig SDK upgrade migration notes (API diffs and gotchas per version).
- [SVELTESTYLE.md](SVELTESTYLE.md) — Svelte 5 code style and component layout (runes, props/callbacks, design tokens).
- [SQLSTYLE.md](SQLSTYLE.md) — Database schema and normalization conventions (star schema design).

## Config Files

- [swiftformat.cfg](swiftformat.cfg) — SwiftFormat configuration for Swift code.
- [swiftlint.yml](swiftlint.yml) — SwiftLint rules for Swift code.
- [zlint.json](zlint.json) — zlint rules for Zig code.
- [Makefile.in](Makefile.in) — the canonical `format`, `lint`, `validate`, and `buildxpin` targets, and `pkg` / `pkginstall` (a macOS `.pkg` built by zigpkg's `zpkg --package`), included by each project's Makefile.
