# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

<!-- One or two sentences: what this is, who uses it, what it does. Then the
     essentials Claude can't infer at a glance: language/runtime versions,
     target platforms (e.g. macOS arm64, Linux arm64/x86_64), and the single
     most important thing to know before touching anything. Keep it to a few
     lines — this is orientation, not a README. -->

## Build / Test

<!-- Exact commands, especially non-obvious ones. Prefer the real invocations
     over wrapper scripts. Cover:
       - build:            e.g. `cmake -B build && cmake --build build -j`
       - run one test:     e.g. `ctest --test-dir build -R <name> -V`
       - full test suite:  e.g. `ctest --test-dir build`
       - lint / format:    e.g. `clang-format -i` / `cargo clippy`
       - codegen step, if any, that must run before build
     Note any required env vars, toolchain versions, or arch-specific flags
     (e.g. -march=native caveats on Apple Silicon vs ARM Linux). -->

## Module layout

<!-- Where the important code lives and what each top-level dir/module owns.
     Point at entry points (main, the primary lib target, the request/event
     loop). List generated dirs and vendored code that must NOT be hand-edited.
     A short annotated tree or bullet-per-dir is plenty. -->

## Architecture conventions

<!-- The patterns to follow so new code matches existing code:
       - error handling (return codes vs exceptions vs Result)
       - memory / ownership / concurrency model
       - naming, file organization, preferred idioms
       - approved dependencies and things to implement directly rather than
         pull in
       - data flow between the major components
     State rules as imperatives: "do X", "never Y". -->

## Project-state heads-ups

<!-- Gotchas that will waste time or break things:
       - flaky tests, known-broken paths, platform-specific quirks
       - files that look editable but are generated
       - steps easy to forget (regenerate X after editing Y)
       - migrations, feature flags, or half-finished refactors in flight -->

## Not yet implemented

<!-- Deliberate gaps, so Claude doesn't assume a feature exists or try to
     "fix" its absence. List stubs, planned-but-missing pieces, and anything
     intentionally out of scope. -->