# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Code style guides — Markdown documents plus the SwiftLint/SwiftFormat configs that enforce them. Docs only: no source code, no build system, no tests, nothing to run.

Consuming projects add this repo as a **git submodule**.

## Build / Test

None. There is no build, test, lint, or format step for this repo.

## Module layout

Flat — every file is at the root.

- `<LANG>STYLE.md` — style guide for a language (`GOSTYLE.md`, `SWIFTSTYLE.md`, `ZIGSTYLE.md`, `SQLSTYLE.md`)
- `<LANG>MIGRATE.md` — version migration notes (`ZIGMIGRATE.md`)
- `CLAUDESTYLE.md`, `READMESTYLE.md` — **templates**, not guides. Skeletons of HTML comments to fill in when authoring a CLAUDE.md or README.md in another project. Follow their section order.
- `swiftformat.cfg`, `swiftlint.yml` — the executable half of `SWIFTSTYLE.md`
- `README.md` — hand-maintained index

## Architecture conventions

- Name new guides `<LANG>STYLE.md` / `<LANG>MIGRATE.md`, uppercase; configs stay lowercase.
- Adding a guide means adding a bullet to the `README.md` index — nothing generates it.
- State rules as imperatives: "do X", "never Y". No hedging, no "consider".
- Keep it terse: rule, one-line rationale, example. No tutorials, no narrative sections.
- Illustrate rules with paired Good/Bad code blocks, matching the existing guides.
- The guides are authoritative. When a rule conflicts with code in some other repo, the code is wrong. Never soften a guide to match existing code.
- A change to a Swift rule must be mirrored in `swiftlint.yml` / `swiftformat.cfg`, and vice versa — they are one standard in two forms.
- Config entries that look like mistakes carry an inline comment explaining why. Preserve those comments; add one when introducing another such entry.

## Project-state heads-ups

- **`swiftlint.yml` `excluded:` globs are `../`-prefixed on purpose.** SwiftLint resolves globs relative to the config file's directory, and this repo is a submodule nested inside the consuming project, so the globs must climb out. A bare `**/.build/**` silently lints hundreds of generated SPM files. Do not "fix" these paths.
- The `/**` contents-glob suffix is likewise deliberate — bare directory globs don't exclude nested `.build/` files on all SwiftLint versions.
- `--disable sortImports` in `swiftformat.cfg` is intentional: `SWIFTSTYLE.md`'s Foundation → Apple → project import order can't be expressed in SwiftFormat, so ordering is left to the author.
- Code in the guides is illustrative. `panic(err.Error())` in the Go templates is scaffolding meant to be replaced downstream — don't correct it in place.
- Guides reference tool versions (Swift 6.0, SwiftFormat 0.61+, Zig 0.16.0). Verify against the actual toolchain before citing one as current.

## Not yet implemented

- No CI, markdown linting, or link checking.
- No LICENSE, CONTRIBUTING, or CODEOWNERS.
- No install/setup instructions for consuming projects.
- Only Go, Swift, Zig, and SQL are covered. Other languages have no guide yet.
