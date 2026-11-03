# WUIDESIGN.md

Authoritative web UI (web console) design rules. Every WUI conforms; TUIDESIGN.md mirrors each shared rule for the terminal. Component code style is SVELTESTYLE.md.

## A. Critical Constraints

### A.1 Forbidden

- A.1.a. Never serve a WUI outside zigwui. Server, guard, live channel, and file browsing come from `../zigwui` — one security model across WUIs.

```zig
const wui = @import("zigwui");          // Good
var server = std.http.Server.init(...); // Bad — own server, own guard
```

- A.1.b. Never write a one-off component in an app when it is reusable. Add it to zigwui's `viewkit`, then import it — the next WUI gets it for free.

```text
zigwui/viewkit/components/DirectoryPicker.svelte   // Good — shared component
myapp/views/src/components/DirectoryPicker.svelte  // Bad  — app-local copy
```

- A.1.c. Never hardcode a color. Use viewkit's design tokens (`viewkit/app.css`) — one palette across WUIs.

```svelte
<span class="text-ink-muted">   <!-- Good -->
<span class="text-[#9aa0a6]">   <!-- Bad — raw hex -->
```

## B. Architecture & File Organization

### B.1 Project Layout

- B.1.a. Three regions, top to bottom: top bar, body, bottom bar. Same frame in every WUI.

```text
 Zig Packager v4.2.1   ~/src/foo                      vault: unlocked   live   <- top bar
 nav │ ...screen...
 saved the recipe                                                            <- bottom bar
```

### B.4 Dependencies & Standard Library

- B.4.a. Depend on `../zigwui` as a path dependency: Zig server from `src/`, page kit from `viewkit/` (components, `api.js`, `live.js`, `app.css`).
- B.4.b. Missing component, route helper, or token: augment zigwui, never work around it in the app.

## C. Code Construction & Logic

### C.1 Logic Layout & Readability

- C.1.a. **Top bar**: one row, pinned. Display name and version first, in `font-header`; version in `font-mono text-ink-muted`.

```svelte
<h1 class="font-header">{state.display_name}</h1><span class="font-mono text-ink-muted">v{state.version}</span>  <!-- Good -->
<h1>myapp</h1>                                                                                                     <!-- Bad — binary name, no version -->
```

- C.1.b. Top bar after the name: app context (working directory, links, mode), truncated first; connection and lock state at the right edge.
- C.1.c. **Bottom bar**: one row, pinned. The current notice (success, problem, running) and status. Same content the TUI's notice row and status line carry.
- C.1.d. Browser tab title: the display name.

```js
$effect(() => { if (state.display_name) document.title = state.display_name; });  // Good
```

### C.2 Naming Conventions

- C.2.a. Display name and version come from the server's state (`display_name`, `version`), which reads the build's `config` module: `build.zig.zon`'s `.description` and `.version`. Never a second copy in the page.

```js
app.state.display_name     // Good
const name = 'Zig Packager'; // Bad — second source
```

## E. Quality, Tooling & Frontend

### E.2 UI Patterns

- E.2.a. Every rule shared with TUIDESIGN.md holds in both. Changing one changes the other.

### E.4 Testing Standards & Test Data

- E.4.a. Verify routes with curl (launch link, cookie jar, `Origin`), the page in a browser. Never assume a render.
