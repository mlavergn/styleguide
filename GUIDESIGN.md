# GUIDESIGN.md

Authoritative native GUI design rules for windows built on ziggui. Every GUI conforms; its frame mirrors TUIDESIGN.md and WUIDESIGN.md, and its code follows ZIGSTYLE.md. Examples use ZigGit's names (`App`, `gui`, `Run`, `Platform`) as the reference app.

## A. Critical Constraints

### A.1 Forbidden

- A.1.a. Never import zigui in an app. Import `ziggui`, and `ziggui_app` in `main.zig` — one kit, one zigui, matching types.

```zig
const ziggui = @import("ziggui");  // Good
const zigui = @import("zigui");    // Bad — bypasses ziggui; a second zigui's `View` does not match
```

- A.1.b. Never import `ziggui_app` outside `main.zig`. The app's core stays headless — every screen tests with no SDL and no window.

```text
app/main.zig            @import("ziggui_app")   // Good — the one SDL file
app/status_screen.zig   @import("ziggui_app")   // Bad  — the screen now needs a window to test
```

- A.1.c. Never write a one-off widget in an app when it is reusable. Add it to ziggui with a gallery section, then use it — the next GUI gets it for free.

```text
ziggui/src/chat.zig   Chip(...)   // Good — shared widget
myapp/app/chip.zig    Chip(...)   // Bad  — app-local copy of a general widget
```

- A.1.d. Never hardcode a color in a view. Read `ziggui.buildTokens()` roles — one palette per theme, light and dark.

```zig
.foreground(ziggui.buildTokens().secondary_label)    // Good
.foreground(ziggui.Color.fromRgb8(122, 122, 122))    // Bad — wrong in dark mode
```

- A.1.e. Never give a sheet or alert a fixed width. It widens to its content's minimum; a fixed width clips a `Picker` that cannot shrink.

```zig
root.alert(shown, form.frameMaxWidth())   // Good — widens to its content
root.alert(shown, form.frameWidth(520))   // Bad  — clips a wide picker at both edges
```

## B. Architecture & File Organization

### B.1 Project Layout

- B.1.a. Three regions, top to bottom: top bar, body, bottom bar (notice row, then status line). Same frame as TUIDESIGN B.1.a and WUIDESIGN B.1.a, but no tabs: the body is a sidebar beside the main view (E.2.b).

```text
 (traffic lights)            ~/src/foo                     <- top bar (macOS)
 REPOSITORIES │ ◉ Uncommitted changes │ commit box         <- sidebar │ main view
 ● foo  main  │   b3a417c v1.5.0 …    │                    
 STASHES      ├───────────────────────┼────────────────    
 WORKTREES    │ files                 │ diff               
   staged 3 paths                                         <- notice row
 s stage · c commit · tab next pane                       <- status line
```

- B.1.b. The GUI is its own module, `app/` (barrel `app/module.zig`, imported as `gui`), beside `cli/` and `web/`. Never import another front end — each is standalone, with its own target and state.
- B.1.c. `main.zig` only connects SDL: forward keys, focus, drops and frames into `gui.App`; fill `gui.Platform`, a struct of function pointers, with SDL calls. Screens call the platform, so tests fake it.

```zig
app.platform.darkFn = Window.dark;          // Good — main fills the hook
if (ziggui_app.colorScheme() == .dark) {}   // Bad  — SDL in a screen
```

- B.1.d. One root struct owns everything, heap-allocated so the pointers callbacks capture stay put. Screens are its fields; a screen's callback reaches the app with `@fieldParentPtr`.

```zig
const app = try App.create(allocator);   // Good — stable address
var app = try App.init(allocator);       // Bad  — moves; captured pointers dangle
```

- B.1.e. The parts that run git share one interface — `init`, `deinit`, `reset`, `shown`, `refresh`, `poll`, `busy`, `key`, `hints` — and give the sidebar their rows or the main view their panes; the two sides compose them, by name, never through a vtable.

### B.4 Dependencies & Standard Library

- B.4.a. Depend on `../ziggui` as a path dependency. Build the runtime with ziggui's `linkApp`, passing the target and optimize mode the dependency was declared with — the same arguments, one zigui.

```zig
const app_mod = @import("ziggui").linkApp(ziggui_dep, .{ .target = target, .optimize = optimize });  // Good
const zigui_dep = b.dependency("zigui", .{});                                                         // Bad — a second zigui
```

- B.4.b. Missing a widget: compose it in ziggui. Missing engine behavior (a view kind, a modifier, a paint primitive, an SDL hook): add it to zigui, SwiftUI-shaped. Never work around either in the app.
- B.4.c. Missing a zigui name in ziggui's barrel: add it there. Never reach through `ziggui.ui`.

```zig
ziggui.ScrollState      // Good
ziggui.ui.ScrollState   // Bad — reaches past the facade
```

## C. Code Construction & Logic

### C.1 Logic Layout & Readability

- C.1.a. **Top bar**: context only — the working directory, centered in the window. The window title and the menu bar already name the app, so no display name or version (a departure from TUIDESIGN C.1.a). On macOS it is the window's drag band; inset both sides by the traffic lights' width, so the text is centered and never under them, and truncate it.

```zig
ziggui.Text(where).truncated().frameMaxWidth().paddingInsets(.{ .leading = lights_w, .trailing = lights_w })   // Good — centered in the window
ziggui.HStack(.{ ziggui.Text(config.display_name), ziggui.Text(config.version), ziggui.Spacer(), ziggui.Text(where) })   // Bad — repeats the title
```
- C.1.b. **Bottom bar**: the notice row (busy mark, last notice) over the status line (key hints, named as the TUI names its keys). Same content as TUIDESIGN C.1.c.
- C.1.c. Compose before extending. Add a zigui view kind only when layout cannot express the widget.

```text
VirtualList: spacer, rows in view, spacer, in a ScrollViewState   // Good — composition
a `.virtual_list` view kind in zigui                               // Bad  — layout already expresses it
```

- C.1.d. Per frame, do per-row work for the rows in view only. Clean data once, where it lands.

```zig
ziggui.VirtualList(Rows, rows, count, row_h, &scroll, Rows.row)   // Good — builds a screenful
ziggui.VStack(allRows(lines))                                      // Bad  — builds 20,000 rows a frame
```

- C.1.e. Build views in the build arena. Content that depends on its own size goes in a `GeometryReader`; anything else reads what the last frame recorded (`ScrollState`), so a headless capture draws two frames.
- C.1.f. Pin `WrappedText` with `.frameAlign(.leading)` after `.frameMaxWidth()`. It centers in a wider frame.

```zig
ziggui.WrappedText(s).frameMaxWidth().frameAlign(.leading)   // Good
ziggui.WrappedText(s).frameMaxWidth()                        // Bad — centered
```

- C.1.g. Decorate text with `.background`, never a flexible shape beside it in a stack. The shape takes the stack's spare width.

```zig
ziggui.Text(title).padding(6).background(fill).cornerRadius(10)               // Good — the tab keeps its title's width
ziggui.VStack(.{ ziggui.Text(title), ziggui.Rectangle(fill).frameHeight(2) }) // Bad  — every tab stretches to share the bar
```

### C.2 Naming Conventions

- C.2.a. Display name per TUIDESIGN C.2.a, shown by the window title and the menu bar, not the top bar (C.1.a). Set the app name and bundle identifier once in `build.zig`'s `config` (`app_name`, `app_identifier`); the bundle, the window title, the desktop entry and the settings folder read them.

```zig
.title = config.app_name,   // Good
.title = "ZigGit",          // Bad — second source
```

- C.2.b. Name a ziggui widget after its SwiftUI equivalent. Invent a name only when SwiftUI has none.

```zig
pub fn HSplitView(...)   // Good — SwiftUI's name
pub fn SplitPane(...)    // Bad  — invented where SwiftUI has one
```

### C.4 Error Handling & Guard Clauses

- C.4.a. Report failure to anything a modal waits on. `begin`/`write` return whether the run started; a refused or unstarted run clears the modal's wait.

```zig
if (!self.job.begin(run)) editor.waiting = false;   // Good
_ = self.job.begin(run);                             // Bad — "Committing…" forever, Esc refused
```

- C.4.b. Give a retry polled every frame an off switch. A failure clears the condition.

```zig
if (at_end and !self.loading and self.more) self.loadNext();   // Good — a failure sets `more` false
if (at_end and !self.loading) self.loadNext();                 // Bad  — retries a refused page at the wake rate
```

- C.4.c. Load settings with `exists()` first: missing is the defaults, silently; malformed is the defaults with a notice. The loader returns null for both.

## D. Systems & State Management

### D.1 Concurrency & Threading

- D.1.a. Run blocking work through a `Job(Run)`. A `Run` is the whole request and its answer: built on the UI thread with every string copied into its own arena, executed on the worker, taken back by `poll`. No locks — the task's done flag orders the writes.
- D.1.b. Compute every arena-backed field first; put `.arena` last. The literal copies the arena before a later field's allocation grows it, so the copy misses the buffer: a leak, then a segfault in `deinit`.

```zig
const path = try arena.allocator().dupe(u8, p);
const run: Run = .{ .path = path, .arena = arena };                                  // Good
const run: Run = .{ .arena = arena, .path = try arena.allocator().dupe(u8, p) };     // Bad — copy misses `path`'s buffer
```

- D.1.c. Stamp every run with the target's epoch. Drop a run that lands after the target moved, whole — the last target's listing never shows on this one.
- D.1.d. Dropping a stale run never drops what queued behind it. Release it, then fall through to the queue.

```zig
if (run.epoch != self.epoch) self.release(run);   // Good — falls through; the queued listing starts
if (run.epoch != self.epoch) return;              // Bad  — the new target's listing never starts
```

- D.1.e. Queue, never block. A listing asked for during a run is queued and started by `poll`; one write may wait behind it; others are refused with a notice.
- D.1.f. Wake by polling. `setBusyInterval` returns 50 ms while any job is busy, 0 otherwise; adopt finished runs in the frame hook. No cross-thread wake — the loop idles with the workers.
- D.1.g. Save settings off the UI thread, newest wins. `main` waits for a save in flight before exiting.

### D.2 Resource Management

- D.2.a. Release an old run only when no live pointer still refers to it. A screen may hold two runs (a listing and a selection's diff) that are one.

```zig
if (old != self.list and old != self.diff) old.release();   // Good
old.release();                                              // Bad — the diff may still be this run; double free
```

- D.2.b. Keep a modal's strings in a modal arena reset on close. Never format a modal's words in the arena that opening it resets — copy them out first.

```zig
const title = try self.allocator.dupe(u8, words); self.closeModal();   // Good — copied out first
self.closeModal(); self.openConfirm(words);                             // Bad  — `words` lived in the arena just reset
```

## E. Quality, Tooling & Frontend

### E.2 UI Patterns

- E.2.a. Every rule shared with TUIDESIGN.md and WUIDESIGN.md holds here — the frame, the key hints — except the top bar's name and version (C.1.a).
- E.2.b. Lay the window out as a sidebar beside one main view, not tabs: the sidebar lists the places (the repositories beside the open one, then its stashes and worktrees), each section a `DisclosureGroup` whose folding is a setting; the main view is the history. Everything shows at once, so a write lists afresh each part it can have changed, never all of them: staging touches the index alone, and a part that runs git once per repository (a scan) waits for a move, a refresh or a write that changes a repository's state, and a focus rescans it only after a minute. View → Hide Sidebar (⌘0) gives the main view the window; `tab` moves through the sidebar and the main view's panes.

```text
sidebar │ history (commits | message, files | diff)   // Good — one window, everything in view
[Status] [Log] [Stash] [Worktrees] [Repos]           // Bad  — five screens, one at a time
```

- E.2.c. Actions are a menu, not a toolbar. Each entry puts what it acts on in place first (the changes selected, the sidebar's keys on a section), then presses its key — menu and key never disagree. On macOS it is the system menu bar. Off macOS the app draws its own title bar in place of the OS's — icon, a `ziggui.MenuBar` (File first: Open Folder…, Settings…, Quit), the path centered, the platform's window buttons — with the same menus and entries. An Alt tap opens File, then the arrow keys, Enter and Esc; below a fixed width the menus fold into one `☰`. The menus are inert under a modal; the window buttons stay live.

```text
[icon] File View Repository Actions Help      ~/src/foo      _ □ ×   // Good — the menu bar in the drawn title bar
[Actions ▾] button at the top bar's right                            // Bad  — a second, partial copy of the menus
```
- E.2.d. Route keys: with a text field focused, consume only Enter, Esc and Tab — characters arrive as a separate text event. With none focused, consume every key the screen uses.
- E.2.e. Turn SDL's unshifted keycodes into what was typed: a shifted letter is its capital. The core decodes `SDLK_*` values without importing SDL.
- E.2.f. ⌘ shortcuts belong to the macOS menu bar; the key handler leaves ⌘ alone there. Off macOS, handle the same shortcuts on Ctrl in the key handler. A ⌘ key a text field needs (⌘A) gets no menu shortcut.
- E.2.g. Multiple selection: the cursor plus a bitset of picked rows, empty when the cursor's row is the selection. Decide toggle or range from `clickCmd`/`clickShift` in the row's tap. Carry picks across a reload by key, never index; act on a selection in one run, one command per batch.
- E.2.h. Draw modals as one `Modal` union in `.alert` on the root. A scrim tap clears the binding and `poll` closes the modal.
- E.2.i. Set an app accent in the theme provider — `accent`, `selection`, `on_accent` — never per view. The accent is a setting: the brand color by default, with a few others to pick in the Settings sheet, saved with the rest. `on_accent` is white, as on macOS — a prominent button is accent with white text — except on an accent where white barely shows (below about 2.5:1: yellow, green), which takes black.

```zig
th.colors.accent = orange; th.colors.on_accent = ziggui.Color.white;   // Good — white on the brand orange
th.colors.accent = yellow; th.colors.on_accent = ziggui.Color.white;   // Bad  — about 1.5:1; black there
```

- E.2.j. Draw the macOS family's dark appearance on true black: override the window, sidebar, secondary, control and glass backgrounds in the theme provider (window `#000000`, controls a step above), never in a view. macOS's stock dark grey reads washed out. Other families keep their own dark.

```zig
if (dark and family == .macos) th.colors.window_background = ziggui.Color.black;   // Good — the provider
.background(ziggui.Color.black)                                                     // Bad  — per view, wrong in light mode
```

- E.2.k. Label the Settings sheet's dismiss button `Done`, with no key in the label; Esc still closes it.

```zig
ziggui.ButtonRoled("Done", .prominent, close)        // Good
ziggui.ButtonRoled("Done  esc", .prominent, close)   // Bad — the key crowds the label
```

- E.2.l. Scroll long-form text sideways, never cut it: a diff line, and the free text that ends a list row (a path, a subject). Lay the list out as wide as its widest row (`ziggui.VirtualListBoth`, `DiffView`), with a horizontal overlay bar. Table values, sidebar rows and the frame's chrome (top bar, notice row, pane titles) truncate. A new selection (file, commit, hunk) returns to the left edge; a refresh keeps the place.

```zig
ziggui.VirtualListBoth(Model, model, count, row_h, &scroll, Model.row, model.widest())   // Good — a long path scrolls into view
ziggui.Text(path).truncated()                                                          // Bad  — the path's end is lost
```

- E.2.m. Put the working tree's changes in the history as a generated first row ("Uncommitted changes · N"), present only while there are changes; selecting it shows the changed paths, their diff with staging, and the commit box in the history's panes. Open on it when there are changes, else on the newest commit; after a commit, select the new commit.
- E.2.n. Write the commit message inline, in the message pane, never in a modal: the box keeps its text while the user stages, moves away and back. An Amend checkbox fills it with the last commit's message unless the user typed their own. A refusal shows under the box and keeps the text; a commit that lands empties it. A quit asks only while the box holds text other than what it started as, and never over another dialog: it waits for that one to close. A late answer to Amend (the last message) is dropped once Amend was unticked.
- E.2.o. Act on a sidebar row's first click: open the repository or worktree, show the stash. Color a repository's name by its state (`added` clean, `removed` dirty, `secondary_label` unknown) with its branch under it; its details (kind, origin, release, version, path) open in a `HoverPopover` after the pointer rests 0.5 s. Mark the list `.popoverPassthrough()`, and keep the loop awake while a hover waits (`HoverState.waiting`).

```zig
ziggui.HoverPopover(row, &sidebar.hover, index, details)   // Good — details on a resting pointer
ziggui.TableRow(&columns, cells, index, selected)          // Bad  — a dashboard table in the sidebar
```

### E.3 Build, Tooling & Task Runners

- E.3.a. Gate everything that links SDL behind `-Dapp`. Core tests always run without it.
- E.3.b. Spell out the SDK for explicit macOS targets: `zigbuildx.Xcode.initFor` resolves it, `linkApp`'s `macos_sdk` takes it.
- E.3.c. Lay out the macOS bundle in the default install step: `<Name>.app/Contents/{MacOS, Info.plist, Resources/<Name>.icns}`, the plist generated from `build.zig.zon`, unsigned. `make app` signs ad hoc; zigpkg signs releases.
- E.3.d. Draw icons as a polygons-only SVG from a committed generator, rendered by zigui's `mkico.py`, then `iconutil`. Commit the outputs — builds need neither tool. `mkico.py` merges same-colored paths, so a knockout over a fill takes its own color.

```text
#010101 knockout on a #000000 tile   // Good — drawn
#000000 knockout on a #000000 tile   // Bad  — merged into the tile
```

- E.3.e. Name the GUI and CLI executables differently by more than case. Off macOS both install to `bin/`; on a case-insensitive disk `ziggit` and `ZigGit` are one file. Fail the build on a collision.
- E.3.f. Link a Linux GUI against glibc, dynamically. SDL loads X11, Wayland and Vulkan at runtime, which a static musl binary cannot.

### E.4 Testing Standards & Test Data

- E.4.a. Test screens headless through `gui.TestApp`: an `App` on a scripted backend, `settle` until idle. Script every external call in order; assert the script finished. Parts that list on workers of their own run in no fixed order: list them one at a time in a test (turn off listing them all on a move), never script a race.
- E.4.b. Ship `--screenshot <out.bmp>` (with `--dark` and a size): the software rasterizer, two frames, no window. `sips -s format png` to read it. Point it at throwaway data.
- E.4.c. Never persist from a screenshot run. Force the appearance; turn saving off.
- E.4.d. After a ziggui or zigui change an app uses, compare the app's screenshots before and after. The rasterizer is deterministic — a refactor matches byte for byte.
- E.4.e. Launch the real window once after a change headless cannot reach — GPU setup, the menu bar, the first frame hook — then quit.
