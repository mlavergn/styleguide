# TUIDESIGN.md

Authoritative terminal UI design rules. Every TUI conforms; WUIDESIGN.md mirrors each shared rule for the web console.

## A. Critical Constraints

### A.1 Forbidden

- A.1.a. Never build on vaxis, zigvaxis, or raw escape codes directly. Use zigtui — one look and one set of widgets across TUIs.

```zig
const tui = @import("zigtui");   // Good
const vaxis = @import("vaxis");  // Bad — bypasses zigtui
```

- A.1.b. Never write a one-off widget in an app when it is reusable. Add it to zigtui, then use it — the next TUI gets it for free.

```text
zigtui/src/drop_zone.zig        // Good — shared widget
myapp/cli/drop_zone.zig         // Bad  — app-local copy of a general widget
```

- A.1.c. Never hardcode a color. Use `tui.Theme` roles — one palette across TUIs.

```zig
.style = tui.Theme.titleText.style()     // Good
.style = .{ .fg = .{ .rgb = .{ 255, 112, 52 } } }  // Bad — raw color
```

## B. Architecture & File Organization

### B.1 Project Layout

- B.1.a. Three regions, top to bottom: top bar, body, bottom bar. Same frame in every TUI.

```text
 Zig Packager v4.2.1   web: http://localhost:8480   ~/src/foo     <- top bar
 ...body...
   saved the recipe                                                <- notice row
 vault unlocked · m menu · q quit · ↑↓ move                        <- status line
```

### B.4 Dependencies & Standard Library

- B.4.a. Depend on `../zigtui` as a path dependency. Shared code stays shared.
- B.4.b. Missing widget or behavior: augment zigtui, never work around it in the app.

## C. Code Construction & Logic

### C.1 Logic Layout & Readability

- C.1.a. **Top bar**: row 0, one line. Display name and version first, in `Theme.titleText`, bold.

```zig
.{ .text = " " ++ config.display_name ++ " v" ++ config.version, .style = title }  // Good
.{ .text = " zpkg" }                                                                 // Bad — binary name, no version
```

- C.1.b. Top bar after the name: app context (links, working directory, mode) in `Theme.secondaryText`. Ellipsize the least important item first.
- C.1.c. **Bottom bar**: last row is `tui.StatusLine` with key hints; the row above it is the notice row.

```zig
const status: tui.StatusLine = .{ .hints = hints };  // Good
try text.print("q quit  m menu", .{});               // Bad — hand-drawn footer
```

### C.2 Naming Conventions

- C.2.a. Display name: `build.zig.zon`'s `.description`. Version: `.version`. Passed through the build's `config` options module — one source for TUI and WUI.

```zig
options.addOption([]const u8, "display_name", zon.description);  // Good
const name = "Zig Packager";                                      // Bad — second source
```

## E. Quality, Tooling & Frontend

### E.2 UI Patterns

- E.2.a. Every rule shared with WUIDESIGN.md holds in both. Changing one changes the other.

### E.4 Testing Standards & Test Data

- E.4.a. Verify layout live in a pty (`script`, fixed `stty cols rows`), render the capture, read the top and bottom bars. Never assume a draw.
