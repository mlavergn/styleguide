# ZIGSTYLE.md

Authoritative Zig code style and project layout conventions. All code must conform — modify the code to match this file, not the reverse.

## A. Critical Constraints

### A.1 Forbidden

- A.1.a. Never use deprecated Zig APIs — cycles are short (sometimes a single dot release); address every deprecation warning in the same change that produced it. See ZIGMIGRATE.md for SDK-version-specific replacements.
- A.1.b. Never hardcode configuration — values that vary between environments, deployments, or users (domains, paths, hostnames, credentials, thresholds) load at runtime through `Config`. Only true compile-time invariants (`is_debug`, struct-internal buffer sizes) are constants in code.

```zig
pub const default_domain: []const u8 = "example.com";  // WRONG — config hardcoded
const domain = cfg.cert_hostname;                              // CORRECT — loaded at runtime
```

## B. Architecture & File Organization

### B.1 Project Layout

- B.1.a. Every directory with multiple `.zig` files has its own `module.zig` barrel (library `src/`, CLI `cli/`, any sub-package).

```zig
src/                   cli/
  module.zig             main.zig      -- Executable entry point
  <name>.zig             module.zig    -- CLI barrel
                         <name>.zig
```

- B.1.b. Module-level compile-time invariants live in `module.zig`: `pub const is_debug = builtin.mode == .Debug;`.

### B.2 File-Level Layout

- B.2.a. **One struct per file** — each file defines exactly one primary public struct; the file name is the struct's role in snake_case (`config.zig` → `Config`, `tls_record.zig` → `TlsRecord`). Supporting types used only by that struct live in the same file above the primary struct, never nested inside it.
- B.2.b. File order, top to bottom: (1) `std` import; (2) scoped logger; (3) module import `const mod = @import("module.zig")`; (4) external dep imports; (5) local type aliases; (6) supporting types; (7) primary public struct; (8) divider comment; (9) unit tests.

```zig
const std = @import("std");
const log = std.log.scoped(.myproject_handler);
const mod = @import("module.zig");

pub const Handler = struct {
    const Self = @This();
    allocator: std.mem.Allocator,
    pub fn init(allocator: std.mem.Allocator, ...) !Self { ... }
    pub fn deinit(self: *Self) void { ... }
};

// -----------------------------------------------------------------------------
// Unit Tests

test "handler" { ... }
```

### B.4 Dependencies & Standard Library

- B.4.a. **No shorthand aliases or namespace shorteners** — always use fully-qualified standard library paths (`std.mem.Allocator`, `std.crypto.*`); this keeps dependencies explicit and code searchable. The only permitted type alias is `const Self = @This();` inside structs.

```zig
const mem = std.mem;                          // WRONG — shortener
const buffer = try std.mem.alloc(u8, 1024);   // CORRECT — explicit path
```

- B.4.b. **Import through the barrel** — import types via the local `module.zig`, never sibling files directly; the import name reflects the package role (`mod` for library, `cli` for CLI). `module.zig` re-exports every public type from its directory plus external deps; the CLI entry point imports both the library and its local CLI barrel.

```zig
const mod = @import("module.zig");            // CORRECT — barrel
const claims_mod = @import("claims.zig");     // WRONG — direct file import
```

- B.4.c. Prefer `std.Io` over `std.posix` — avoid `std.posix` unless irreplaceable, reaching for the corresponding `std.Io` API first.

## C. Code Construction & Logic

### C.1 Logic Layout & Readability

- C.1.a. **Struct-as-namespace** — all behavior is namespaced on structs; no free-standing public functions aside from trivial inline helpers. Every struct opens with `const Self = @This();` (the only permitted alias). Fields first, before functions, grouped with `// comment` headers when numerous. Functions are methods taking `self: *Self` (or `*const Self` read-only); static methods still live inside the struct. Function order: `init`, `deinit`, `load`/`unload`, public API, private helpers, serialization.

```zig
pub const State = struct {
    const Self = @This();
    arena: std.heap.ArenaAllocator,
    config: *const mod.Config,
    // Buffers
    input: []f32 = undefined,

    pub fn load(self: *Self) !void { ... }
    pub fn describe(self: *const Self) void { ... }
};
```

- C.1.b. **Pointer vs value semantics** — `init` returns `Self` by value (the caller stores it). Mutable methods take `*Self`, read-only methods `*const Self`. Shared read-only data that outlives the struct is borrowed and stored as a const pointer (`config: *const mod.Config`).
- C.1.c. **Default field values** — give fields defaults where a sensible zero/undefined value exists, keeping `init` focused on essentials. Fields populated later (after `load` or assignment) use `= undefined` as a sentinel, guarded by the `loaded` flag.

```zig
loaded: bool = false,
threshold: f32 = 1.0,
config: *const mod.Config = undefined,
buffer: []f32 = undefined,
```

### C.2 Naming Conventions

| Element                     | Convention            | Example                                   |
| --------------------------- | --------------------- | ----------------------------------------- |
| Struct / Enum / Union       | PascalCase            | `Config`, `RequestHandler`, `MessageType` |
| Enum variant                | camelCase             | `.streaming`, `.startOfBlock`             |
| Function                    | camelCase             | `init`, `deinit`, `processItem`           |
| Field / Constant / Variable | snake_case            | `max_retries`, `is_debug`, `num_items`    |
| Type alias                  | PascalCase            | `const Self = @This()`                    |
| Log scope                   | `.<project>_<module>` | `.myproject_parser`                       |

### C.3 Control Flow & Branching

- C.3.a. **Performance annotations** — `inline` for performance-critical small functions (`pub inline fn toIndex(...) u32`); `@branchHint(.likely)` / `@branchHint(.unlikely)` on hot-path branches; `@prefetch(self.data.ptr, .{ .rw = .read, .cache = .data, .locality = 3 })` for data known to be needed soon.

### C.4 Error Handling & Guard Clauses

- C.4.a. Fallible functions return `!T`; pure computations and destructors return plain types (`deinit ... void`). Errors propagate with `try` by default; use `catch` to log before re-returning, and to clean up a partially-initialized arena on init failure.

```zig
const handle = Resource.init(self.allocator, path) catch |err| {
    log.err("Failed to open: {s} [{any}]", .{ path, err });
    return err;
};
self.load() catch |err| { self.arena.deinit(); return err; };
```

## D. Systems & State Management

### D.1 Concurrency & Threading

- D.1.a. **Io ownership** — `std.Io` is a first-class capability flowing through networking, filesystem, randomness, and entropy-consuming crypto; treat it like the allocator. Io is the second parameter — immediately after `self` (or after `allocator` in `init`). Store it as a field when the struct uses it across its lifetime; pass it per-call for stateless helpers. When a struct is itself an I/O surface (TLS connection, codec, framed channel), expose direct named methods (`read`, `writeAll`, `print`) rather than handing back an `Io.Reader`/`Writer` wrapper.

```zig
pub fn init(allocator: std.mem.Allocator, io: std.Io, stream: std.Io.net.Stream,
    identity: struct { cert_chain: []const []const u8, priv_key: mod.PrivateKey }) Self { ... }
```

### D.2 Resource Management

- D.2.a. **Init / Deinit lifecycle** — `init` returns `!Self`, constructing a local value and returning it (`const self` when no field needs post-literal mutation, `var self` otherwise). `deinit` takes `*Self`, never fails, cleans up child resources in reverse init order, then frees owned pointers (`_ = self;` when nothing to free). Structs managing heavy resources separate one-time construction (`init`) from data loading (`load`) with a `loaded: bool` guard; `deinit` calls `unload` then frees the allocator.

```zig
pub fn init(allocator: std.mem.Allocator, config: *const mod.Config) !Self {
    var self = Self{ .arena = std.heap.ArenaAllocator.init(allocator), .config = config };
    self.allocator = self.arena.allocator();
    return self;
}

pub fn deinit(self: *Self) void {
    self.child_b.deinit();   // children first, reverse order
    self.child_a.deinit();
    self.allocator.destroy(self.owned_ptr);
}
```

- D.2.b. **Allocator ownership** — the parent's allocator is always the first parameter to `init`, stored as a field; after init use `self.allocator`, never re-asking for one in helpers. A **direct allocator** frees each allocation in `deinit`; an **arena** routes all allocations through it and frees everything with one `arena.deinit()` (preferred for many same-lifetime allocations). Owned heap pointers use `create`/`destroy`. `main` pulls its allocator and `io` from the runtime-supplied `std.process.Init`; library/test code falls back to `DebugAllocator`.

```zig
fn parseEntry(allocator: std.mem.Allocator, dict: SomeDict) !Entry { ... }  // WRONG — re-asks
fn parseEntry(self: *Self, dict: SomeDict) !Entry {                          // CORRECT
    const buf = try self.allocator.alloc(u8, ...);
}
```

- D.2.c. **Defer** — pair every resource with `defer` at acquisition; multiple defers run LIFO, matching reverse cleanup order.

```zig
var handler = try mod.Handler.init(allocator);  defer handler.deinit();
var file = try std.fs.cwd().createFile(name, .{});  defer file.close();
const buffer = try allocator.alloc(u8, size);  defer allocator.free(buffer);
```

### D.3 Logging, Diagnostics & Telemetry

- D.3.a. **Logging** — every file declares a scoped logger tagged `.<project>_<module>` (`std.log.scoped(.myproject_parser)`). Functions trace entry with `@src()`: `log.debug("{s}:{d} :: {s}", .{ @src().file, @src().line, @src().fn_name })`. The executable configures level and log function via `std_options`.

```zig
pub const std_options: std.Options = .{ .log_level = .debug, .logFn = mylib.Logger.logFn };
```

- D.3.b. **Serialization (debug dumps)** — structs needing debug dumping implement `jsonStringify` for custom JSON output plus a `serialize` convenience method writing to a file.

```zig
pub fn jsonStringify(self: *const Self, writer: anytype) !void {
    try writer.beginObject();
    try writer.objectField("name");  try writer.write(self.name);
    try writer.endObject();
}
```

## E. Quality, Tooling & Frontend

### E.4 Testing Standards & Test Data

- E.4.a. Tests sit at the bottom of the file after a divider (`// ---` then `// Unit Tests`). Each test uses `std.testing.allocator`; sets up dependencies with a `defer deinit()` for each; exercises the struct; asserts with `std.testing.expectEqual` / `expectEqualStrings`. Tests initialize the full dependency chain rather than mocking; shared fixtures (paths, constants) live in `module.zig`.

```zig
test "parser" {
    const allocator = std.testing.allocator;
    var config = try mod.Config.init(allocator);  defer config.deinit();
    var parser = try mod.Parser.init(allocator, &config);  defer parser.deinit();
    const result = try parser.parse("input data");
    try std.testing.expectEqual(42, result.count);
}
```

- E.4.b. `module.zig` uses `refAllDecls` so re-exported modules have their tests discovered: `test { std.testing.refAllDecls(@This()); }`.
