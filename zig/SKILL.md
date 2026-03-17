---
name: zig
description: "Write correct, idiomatic Zig 0.15 code. Use when writing new Zig code, editing existing Zig files, debugging Zig compilation errors, reviewing Zig code, or working with build.zig files. Triggers on any task involving .zig files. Critical: LLM training data contains outdated Zig patterns (0.11-0.13) that will produce broken code — this skill provides the current 0.15 patterns."
---

# Zig 0.15

LLM training data is based on Zig 0.11-0.13. Zig 0.15 has massive breaking changes. ALWAYS consult [references/zig-0.15-migration.md](references/zig-0.15-migration.md) before writing Zig code.

## Tools

Always available — use these before guessing at APIs:

- **`zigdoc`** — Discover current APIs for std and third-party deps. Use before writing any code involving unfamiliar APIs.
  ```bash
  zigdoc std.fs
  zigdoc std.posix.getuid
  zigdoc vaxis.Window
  ```
- **`ziglint`** — Lint Zig code for style and correctness issues.

## Critical Changes You Will Get Wrong

### std.io → std.Io (the I/O overhaul)

`std.io.getStdOut().writer()` is GONE. The entire I/O system was redesigned:

```zig
// WRONG (old):
const stdout = std.io.getStdOut().writer();
try stdout.print("hello\n", .{});

// RIGHT (0.15):
var buf: [4096]u8 = undefined;
var writer = std.fs.File.stdout().writer(&buf);
defer writer.interface.flush() catch {};
try writer.interface.print("hello {s}\n", .{"world"});
```

- `std.io.Writer` → `std.Io.Writer`
- `std.io.Reader` → `std.Io.Reader`
- `std.io.AnyWriter`/`AnyReader`/`GenericWriter`/`GenericReader` → `std.Io.Writer`/`std.Io.Reader`
- `std.io.BufferedWriter`/`BufferedReader` → REMOVED (buffering is in the interface now)
- Functions accepting writers: use `*std.Io.Writer` parameter type
- ALL buffered output requires explicit `flush()`

**Allocating writer** (writes to allocated memory):
```zig
var writer: std.Io.Writer.Allocating = .init(allocator);
defer writer.deinit();
try writer.writer.print("hello {s}", .{"world"});
const output = try writer.toOwnedSlice();
```

**JSON writing:**
```zig
var buf: [4096]u8 = undefined;
var writer = std.fs.File.stdout().writer(&buf);
defer writer.interface.flush() catch {};

var jw: std.json.Stringify = .{
    .writer = &writer.interface,
    .options = .{ .whitespace = .indent_2 },
};
try jw.write(my_struct);
```

### ArrayList and HashMap — allocator per call

```zig
// WRONG (old):
var list = std.ArrayList(i32).init(allocator);
try list.append(42);

// RIGHT (0.15) — initialize with .empty, pass allocator to every mutating op:
var list: std.ArrayList(u32) = .empty;
defer list.deinit(allocator);
try list.append(allocator, 42);
```

**HashMap/StringHashMap** (same pattern — default to unmanaged):
```zig
var map: std.StringHashMapUnmanaged(u32) = .empty;
defer map.deinit(allocator);
try map.put(allocator, "key", 42);
```

Initialize with `.empty`, NOT `.init()` or `.{}`.

### Build system

```zig
b.addExecutable(.{
    .name = "foo",
    .root_module = b.createModule(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    }),
});
```

- `addStaticLibrary()` → `addLibrary(.{ .linkage = .static })`
- Use `b.createModule()` for root modules
- `.root_source_file = .{ .path = "..." }` → `.root_source_file = b.path("...")`
- Path strings → `LazyPath`: `.{ .cwd_relative = path }`

### Type reflection tags are lowercase

```zig
// WRONG: .Int, .Float, .Struct, .Enum
// RIGHT: .int, .float, .@"struct", .@"enum"
```

### Cast builtins are single-argument

All casts infer return type from context. Do NOT pass destination type:
```zig
// WRONG: @ptrCast(DestType, ptr)
// RIGHT: const result: DestType = @ptrCast(ptr);
```

Same for `@bitCast`, `@truncate`, `@floatFromInt`, `@intFromFloat`, `@enumFromInt`, `@ptrFromInt`.

### Other renames

- `@intToPtr`→`@ptrFromInt`, `@ptrToInt`→`@intFromPtr`
- `@intToFloat`→`@floatFromInt`, `@floatToInt`→`@intFromFloat`
- `@intToEnum`→`@enumFromInt`, `@enumToInt`→`@intFromEnum`
- `@errSetCast`→`@errorCast`
- `@fieldParentPtr(T, "f", ptr)` → `@fieldParentPtr(ptr, .field_name)`
- `@setCold(true)` → `@branchHint(.cold)`
- `@export(fn, ...)` → `@export(&fn, ...)`
- `std.os` → `std.posix`
- `std.rand` → `std.Random`
- `std.TailQueue` → `std.DoublyLinkedList`
- `callconv(.C)` → `callconv(.c)`
- `async`/`await` → REMOVED

## Zig Style

- `camelCase` for functions/methods
- `snake_case` for variables, parameters, constants
- `PascalCase` for types, structs, enums
- Prefer `const foo: Type = .{ .field = value };` over `const foo = Type{ .field = value };`
- File order: `//!` module doc comment, `const Self = @This();`, imports, `const log = std.log.scoped(...)`
- Pass allocators explicitly; use `errdefer` for cleanup on error
- Keep tests inline with the code they cover
- Comments explain why, not what

## Before Writing Zig Code

1. Run `zigdoc` to verify any std library API you're unsure about
2. Read existing code in the project first — match established patterns
3. Check project's style guide if one exists
4. After writing, verify with `zig build test` — don't guess

## Full Reference

See [references/zig-0.15-migration.md](references/zig-0.15-migration.md) for complete migration details with all code examples, the new Writer/Reader VTable layout, signal handling patterns, format string changes, and allocator API updates.
