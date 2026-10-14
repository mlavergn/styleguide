# MIGRATE.md

Migration notes for Zig SDK upgrades. Format: in A we did B, in Z we do Y. Entries stay tight — show the diff, name the symptom, flag non-obvious gotchas; no prose tutorials.

## 0.15.2 to 0.16.0

### `std.zig.system.resolveTargetQuery` — adds `io`

```zig
self.target = try std.zig.system.resolveTargetQuery(query);       // 0.15.2
self.target = try std.zig.system.resolveTargetQuery(io, query);   // 0.16.0
```

### `std.zig.system.darwin.getSdk` — adds `io` (after allocator)

```zig
self.sdk = std.zig.system.darwin.getSdk(allocator, &self.target);       // 0.15.2
self.sdk = std.zig.system.darwin.getSdk(allocator, io, &self.target);   // 0.16.0
```

Obtain `std.Io` in `build.zig` via `const io = b.graph.io;`.

### `linkFramework` / `linkLibC` / `linkSystemLibrary` / `linkLibrary` — `Compile` → `Module`

`linkLibC()` is gone; set `link_libc = true` in module options. `linkFramework` now takes a `LinkFrameworkOptions` struct. `linkLibrary` (linking another Zig artifact) moves to the consumer module, called before constructing the `Compile` step. Symptoms: `no field or member function named 'linkFramework'/'linkLibrary' in 'Build.Step.Compile'`.

```zig
// 0.15.2
const lib = b.addLibrary(.{ .name = name, .root_module = module, .linkage = .static });
lib.linkFramework("CoreFoundation");
lib.linkLibC();
const exe = b.addExecutable(.{ .name = name, .root_module = exe_mod });
exe.linkLibrary(lib);

// 0.16.0
const module = b.addModule(name, .{ .root_source_file = src, .target = target, .optimize = optimize, .link_libc = true });
module.linkFramework("CoreFoundation", .{});
const lib = b.addLibrary(.{ .name = name, .root_module = module, .linkage = .static });
exe_mod.linkLibrary(lib);
const exe = b.addExecutable(.{ .name = name, .root_module = exe_mod });
```

### `main` takes `std.process.Init`; `argsAlloc`/`argsFree` removed

`Init` carries preinitialized `args`, `environ`, `gpa`, `io`, `arena` (`Init.Minimal` is the smaller `args`/`environ` variant). `init.arena` is runtime-set and freed at process exit — prefer it over a private `ArenaAllocator`; use `init.gpa` for heap freed mid-process. Zero-alloc iterator for `std.process.args()` is `init.args.iterate()` / `init.minimal.args.iterate()`, same `.next()`/`.skip()`.

```zig
// 0.15.2
pub fn main() !void {
    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);
}
// 0.16.0
pub fn main(init: std.process.Init) !void {
    const io = init.io;
    const args = try init.minimal.args.toSlice(init.arena.allocator());
}
```

### `std.process.Init` parameter shadows `init()` method

In a struct method taking `Init`, name the parameter `process_init` (project precedent) or hit `error: function parameter shadows declaration of 'init'`. Top-level `pub fn main(init: ...)` is fine since `main` isn't a struct method.

```zig
pub fn run(self: *Self, process_init: std.process.Init) void { const io = process_init.io; }
```

### `std.fs.cwd()` / `std.fs.File` → `std.Io.Dir.cwd()` / `std.Io.File`

All file ops require `io`; field types change too (`file: std.fs.File` → `std.Io.File`).

```zig
// 0.15.2
const file = try std.fs.cwd().openFile(path, .{ .mode = .read_write });
defer file.close();
const data = try file.read(buf);
try file.write(buf);
// 0.16.0
const file = try std.Io.Dir.cwd().openFile(io, path, .{ .mode = .read_write });
defer file.close(io);
const data = try file.readStreaming(io, &.{buf});       // buffer is [][]u8
_ = try file.writeStreaming(io, &.{buf});               // buffer is [][]const u8
```

### `std.fs.File.stdout()` / `stderr()` / `stdin()` → `std.Io.File.*`

Buffered writer construction takes `io` plus a buffer; the `Writer` interface is the `.interface` field. One-shot unbuffered: `std.Io.File.stdout().writeStreamingAll(io, bytes)`.

```zig
var stdout = std.fs.File.stdout().writer(&buf);          // 0.15.2
var stdout = std.Io.File.stdout().writer(io, &buf);      // 0.16.0
try stdout.interface.print("hello {s}\n", .{name});
try stdout.interface.flush();
```

### `readFileAlloc` — new signature, `Io.Limit` for size

```zig
const data = try std.fs.cwd().readFileAlloc(allocator, path, 1024 * 1024);                  // 0.15.2
const data = try std.Io.Dir.cwd().readFileAlloc(io, path, allocator, .limited(1024*1024));  // 0.16.0
```

### `createFile` / `writeAll`

```zig
// 0.15.2
const file = try std.fs.cwd().createFile(path, .{});
defer file.close();
try file.writeAll(buf);
// 0.16.0
const file = try std.Io.Dir.cwd().createFile(io, path, .{});
defer file.close(io);
try file.writeStreamingAll(io, buf);
```

### `makePath` → `createDirPath`

Renamed and adds `io`. `createDirPathStatus` reports whether the path was newly created.

```zig
try std.fs.cwd().makePath("a/b/c");                  // 0.15.2
try std.Io.Dir.cwd().createDirPath(io, "a/b/c");     // 0.16.0
```

### `CreateFileOptions.mode` → `permissions: File.Permissions`

Defaults `.default_file` (0o666) / `.default_dir` (0o777) cover common cases; `.fromMode` only for non-default bits.

```zig
const file = try std.fs.cwd().createFile(path, .{ .mode = 0o600 });                              // 0.15.2
const file = try std.Io.Dir.cwd().createFile(io, path, .{ .permissions = .fromMode(0o600) });    // 0.16.0
```

### `std.net` removed — networking via `std.Io.net`

Whole namespace gone. `std.net.Address` splits into `std.Io.net.IpAddress` (tagged union of `Ip4Address`/`Ip6Address`) and `std.Io.net.UnixAddress`. `std.posix.socket`/`bind`/`listen`/`accept`/`connect`/`send`/`recv`/`shutdown`/`close` are also gone — connection-shaped ops route through `std.Io.net`; `std.posix.setsockopt`, `getpeername`, and `IPPROTO`/`TCP`/`SOL`/`SO` constants survive.

**Addresses** — `Ip4Address` is `{ bytes: [4]u8, port: u16 }`, no `sockaddr` to hand to posix:

```zig
const addr = try std.net.Address.parseIp("127.0.0.1", 8080);    // 0.15.2
const ua = try std.net.Address.initUnix("/tmp/sock");
const ip: std.Io.net.IpAddress = try .parse("127.0.0.1", 8080); // 0.16.0
const ua = try std.Io.net.UnixAddress.init("/tmp/sock");
```

**Server** — produced by `address.listen(io, ...)`; `accept` returns `Stream` directly (no more `Server.Connection`); `deinit`/`close` take `io`. `UnixAddress.listen` has its own `ListenOptions` (no `reuse_address`).

```zig
// 0.15.2
var server = try addr.listen(.{ .reuse_address = true });
defer server.deinit();
const conn = try server.accept();
defer conn.stream.close();
// 0.16.0
var server = try ip.listen(io, .{ .reuse_address = true, .kernel_backlog = backlog });
defer server.deinit(io);
const stream = try server.accept(io);
defer stream.close(io);
```

**Client**

```zig
// 0.15.2
const fd = try std.posix.socket(addr.any.family, std.posix.SOCK.STREAM, std.posix.IPPROTO.TCP);
try std.posix.connect(fd, &addr.any, addr.getOsSockLen());
defer std.posix.close(fd);
// 0.16.0
const stream = try ip.connect(io, .{ .mode = .stream });
defer stream.close(io);
try stream.shutdown(io, .send);
```

**Reading/writing a `Stream`** — idiomatic: `stream.reader(io, buf)` / `stream.writer(io, buf)`, whose `.interface` is a `std.Io.Reader`/`Writer`:

```zig
var read_buf: [4096]u8 = undefined;
var r = stream.reader(io, &read_buf);
const n = try r.interface.readSliceShort(dest);
var write_buf: [4096]u8 = undefined;
var w = stream.writer(io, &write_buf);
try w.interface.writeAll(payload);
try w.interface.flush();
```

If you own framing/buffering, drop to the vtable. `netRead` returns 0 on EOF; `netWrite` may return short, so loop. `data` is `[]const []const u8`; `splat` repeats the last slice (pass `1` for "send once").

```zig
var iov: [1][]u8 = .{buffer};
const n = try io.vtable.netRead(io.userdata, stream.socket.handle, &iov);
var offset: usize = 0;
while (offset < data.len) {
    const w = try io.vtable.netWrite(io.userdata, stream.socket.handle, &.{}, &.{data[offset..]}, 1);
    if (w == 0) return error.ConnectionClosed;
    offset += w;
}
```

For benchmarks where buffering distorts measurements, drop to libc — `stream.socket.handle` is still `std.posix.fd_t`. Requires `link_libc = true` (always on macOS, explicit on Linux/musl/Windows).

```zig
const sent = std.c.send(stream.socket.handle, buffer.ptr, buffer.len, 0);
const got  = std.c.recv(stream.socket.handle, buffer.ptr, buffer.len, 0); // 0 = EOF, <0 = error
```

**Socket options not exposed by `std.Io.net`** — `ConnectOptions`/`ListenOptions` carry only `mode`, `protocol`, `kernel_backlog`, `reuse_address`, `timeout`. For `TCP_NODELAY`, `SO_RCVBUF`, `SO_KEEPALIVE`, etc., set via `std.posix.setsockopt` on `stream.socket.handle` after connect (fine for `TCP_NODELAY` since it only affects subsequent writes).

```zig
try std.posix.setsockopt(stream.socket.handle, std.posix.IPPROTO.TCP, std.posix.TCP.NODELAY, &std.mem.toBytes(@as(c_int, 1)));
```

### `std.Thread.sleep` → `io.sleep`

Cancelable; `Clock` selects time source (`.awake` ≈ old monotonic); returns `Cancelable!void`. Duration helpers: `std.Io.Duration.{fromNanoseconds, fromMicroseconds, fromMilliseconds, fromSeconds}`. Any struct that sleeps must store `io: std.Io` and accept it in `init` — typically cascades up the ownership chain.

```zig
std.Thread.sleep(10 * std.time.ns_per_ms);          // 0.15.2
try io.sleep(.fromMilliseconds(10), .awake);        // 0.16.0
```

### `std.time.nanoTimestamp` / `timestamp` / `milliTimestamp` → `std.Io.Clock`

`.real` for wall time, `.awake` for monotonic; `Timestamp.nanoseconds` is `i96`. `start.untilNow(io, .awake)` returns a `Duration` directly.

```zig
const start = std.time.nanoTimestamp();                              // 0.15.2
const elapsed_ns = std.time.nanoTimestamp() - start;
const start = std.Io.Clock.awake.now(io);                            // 0.16.0
const end   = std.Io.Clock.awake.now(io);
const elapsed_ns: i128 = @as(i128, end.nanoseconds) - @as(i128, start.nanoseconds);
const now_s: i64 = @intCast(@divTrunc(std.Io.Clock.real.now(io).nanoseconds, std.time.ns_per_s)); // Unix epoch secs
```

### Tests: `std.testing.io`

```zig
test "open device" {
    const result = Serial.init(std.testing.allocator, std.testing.io, "/dev/cu.nonexistent");
    try std.testing.expectError(error.FileNotFound, result);
}
```

### `std.testing.refAllDeclsRecursive` → `refAllDecls`

Drop-in for barrel-style `module.zig` since each re-exported file has its own `refAllDecls`. If you relied on recursive discovery from a single root, add `refAllDecls` to each intermediate barrel.

### `ArrayList(u8).writer(allocator)` → `std.Io.Writer.Allocating`

Owns its buffer; pass `&aw.writer` to writer-consuming APIs. Field/method swaps: `output.items` → `output.written()`, `output.items.len` → `output.written().len`; allocator-owning variants drop the `allocator` param (`ensureTotalCapacity`, `toOwnedSlice`); `clearRetainingCapacity` unchanged.

```zig
// 0.15.2
var output: std.ArrayList(u8) = .empty;
errdefer output.deinit(allocator);
const writer = output.writer(allocator);
try writer.writeAll(bytes);
return output.toOwnedSlice(allocator);
// 0.16.0
var output: std.Io.Writer.Allocating = .init(allocator);
errdefer output.deinit();
try output.writer.writeAll(bytes);
return output.toOwnedSlice();
```

### `std.io.null_writer` → `std.Io.Writer.Discarding`

Real `Writer` that drops everything; needs a small scratch buffer for the vtable.

```zig
try interpret(content, std.io.null_writer);          // 0.15.2
var disc_buf: [64]u8 = undefined;                    // 0.16.0
var disc: std.Io.Writer.Discarding = .init(&disc_buf);
try interpret(content, &disc.writer);
```

### `Writer.writeByteNTimes` → `splatByteAll`

`splatBytesAll(bytes, count)` for multi-byte patterns.

```zig
try writer.writeByteNTimes(' ', count);   // 0.15.2
try writer.splatByteAll(' ', count);      // 0.16.0
```

### `std.ArrayList(T) = .{}` → `.empty`

Applies to field defaults and re-assignment after `deinit`. Compiler error is misleading — complains about missing `items`/`capacity` rather than the deprecation; recognize by the `std.ArrayList` reference on the source line.

```zig
sessions: std.ArrayList(Session) = .{},      // 0.15.2
sessions: std.ArrayList(Session) = .empty,   // 0.16.0
```

### `std.ArrayList` is unmanaged — methods take `allocator` per call

`ArrayList(T).init(allocator)` is gone; the list no longer stores its allocator, so `append`, `appendSlice`, `deinit`, `toOwnedSlice`, `ensureTotalCapacity`, etc. take the allocator as their first argument. Pair with `.empty`. Symptom: `error: expected 2 argument(s), found 1` on old call sites. `clearRetainingCapacity` is still allocator-free.

```zig
// 0.15.2
var list = std.ArrayList(Sphere).init(allocator);
defer list.deinit();
try list.append(item);
return list.toOwnedSlice();
// 0.16.0
var list: std.ArrayList(Sphere) = .empty;
defer list.deinit(allocator);
try list.append(allocator, item);
return list.toOwnedSlice(allocator);
```

### `std.heap.GeneralPurposeAllocator` → `std.heap.DebugAllocator`

In `main`, prefer `init.gpa` from `process.Init`; reach for `DebugAllocator` directly only when you can't take an `Init` (e.g. library code under tests with custom setup).

```zig
var gpa = std.heap.GeneralPurposeAllocator(.{}){};      // 0.15.2
var gpa: std.heap.DebugAllocator(.{}) = .init;          // 0.16.0
```

### `std.process.Child.init` / `spawnAndWait` → `std.process.spawn` + `child.wait`

Spawning is an `Io` op. `Term` is now a tagged union (`exited`/`signal`/`stopped`/`unknown`).

```zig
// 0.15.2
var child = std.process.Child.init(&.{ "mutool", "convert", path }, allocator);
child.stdout_behavior = .Ignore;
child.stderr_behavior = .Ignore;
const term = try child.spawnAndWait();
if (term.Exited == 0) { /* ok */ }
// 0.16.0
var child = try std.process.spawn(io, .{ .argv = &.{ "mutool", "convert", path }, .stdout = .ignore, .stderr = .ignore });
const term = try child.wait(io);
if (term == .exited and term.exited == 0) { /* ok */ }
```

### `std.process.getEnvVarOwned` → `Environ.getAlloc`

Returns `error.EnvironmentVariableMissing` when unset (as before). `Environ` is small/cheap — pass by value when env access is needed outside `main`.

```zig
const home = try std.process.getEnvVarOwned(allocator, "HOME");           // 0.15.2
const home = try init.minimal.environ.getAlloc(allocator, "HOME");        // 0.16.0
```

### `std.process.getCwdAlloc` → `std.process.currentPathAlloc(io, gpa)`

`currentPath(io, &buffer)` writes into a caller-provided buffer.

```zig
const cwd = try std.process.getCwdAlloc(allocator);              // 0.15.2
const cwd = try std.process.currentPathAlloc(io, allocator);     // 0.16.0
```

### `std.posix.PROT.READ` on Darwin — packed struct field, not a constant

On macOS/iOS `std.posix.PROT` is `std.macho.vm_prot_t` (packed struct); other platforms are also packed bool structs, so the new syntax works everywhere.

```zig
try std.posix.mmap(null, size, std.posix.PROT.READ, .{ .TYPE = .PRIVATE }, fd, 0);   // 0.15.2
try std.posix.mmap(null, size, .{ .READ = true }, .{ .TYPE = .PRIVATE }, fd, 0);     // 0.16.0
```

### `std.crypto.random` → `Io.random` / `Io.randomSecure`

Process-global namespace removed. `Io.random` is fast/seeded/threadsafe; `Io.randomSecure` returns `RandomSecureError!void` and pulls fresh OS entropy (use for keys, nonces, salts). Crypto helpers that previously pulled from `std.crypto.random` internally now take `io` — e.g. `std.crypto.sign.Ed25519.KeyPair.generate(io)`; in tests use `std.testing.io`.

```zig
const n = std.crypto.random.int(u64);    // 0.15.2
var b: [8]u8 = undefined;                 // 0.16.0
io.random(&b);
const n = std.mem.readInt(u64, &b, .little);
var key: [32]u8 = undefined;
try io.randomSecure(&key);
```

**Process-internal unique-id escape hatch** — when the only reason to call `std.crypto.random.int(u64)` was a process-unique handle (session/request id; uniqueness, not unpredictability), an atomic counter avoids threading `io` everywhere. For tokens or anything observable by an attacker, plumb `io` and use `randomSecure`.

```zig
var session_counter: std.atomic.Value(u64) = .init(0);
const sid = session_counter.fetchAdd(1, .monotonic);
```

### FFI boundaries — process-global single-threaded `Io`

C-exported and WASM-exported functions can't take `Io` in their signatures but call library code that requires it. Use the runtime's process-global single-threaded instance — no concurrency or cancelation, but fine for synchronous file I/O, mmap, and process spawning behind FFI.

```zig
fn ffiIo() std.Io { return std.Io.Threaded.global_single_threaded.io(); }

export fn lib_open(path_ptr: [*:0]const u8) ?*Handle {
    const doc = MyLib.open(ffiIo(), c_allocator, std.mem.span(path_ptr)) catch return null;
    return @ptrCast(doc);
}
```

## 0.14.x to 0.15.2

### `std.io` → `std.Io`

Lowercase namespace deprecated in 0.15.x, removed entirely in 0.16.0.

```zig
const r = std.io.Reader;   // pre-0.15.2
const r = std.Io.Reader;   // 0.15.2+
```
