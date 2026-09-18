# Zig Style Guide

This guide defines Zig specific conventions for our scientific and engineering software.

**Read the Performance Oriented Software Guide first.** That guide defines the shared principles for correctness, Data Oriented Design, memory behaviour, ownership, allocation, SIMD, threading, API design, numerical accuracy, and benchmarking. This guide explains how we express those principles idiomatically in Zig.

Our priorities remain:

1. **Make it correct.**
2. **Make it fast.**
3. **Make it simple for users.**

---

## Quick Rules for New Contributors

- follow the Performance Oriented Software Guide;
- put correctness first and test isolated behaviour locally;
- use `std.testing.allocator` for allocating tests unless the test requires a different allocator;
- test every deliberately returned error condition;
- use slices and constness to express borrowed data and mutation;
- prefer enums, optionals, and tagged unions to loosely coupled flags;
- choose compile time or runtime dispatch deliberately;
- use Zig errors for legitimate runtime failure;
- use `std.debug.assert` for programmer errors and internal invariants;
- use `unreachable` only for states impossible by construction;
- pass allocators explicitly to functions that allocate, resize, or free;
- do not store allocators in application types;
- use `outer_alloc` for memory that leaves a function and `local_alloc` for scratch memory;
- use `defer` and `errdefer` to keep cleanup local;
- functions that perform I/O should take a `std.Io`;
- prefer the `std.Io` interface for concurrency unless direct `std.Thread` control is genuinely required;
- use `comptime`, `inline`, and `@Vector` deliberately recognising the tradeoffs associated with them;
- keep internal imports direct and public reexports deliberate;
- expose a small C ABI using C compatible types;
- use normal Zig style and let `zig fmt` handle mechanical formatting.

---

## 1. Testing and Verification

> **Principle:** Write code so that isolated behaviour is easy to test, and test failures as deliberately as successes.

Functions with isolated deterministic behaviour should normally have local `test` blocks. Put test blocks at the bottom of the file that declares the code being tested. The normal file order is therefore public API and types, implementation functions in approximate call order, then tests.

Use `std.testing.allocator` for allocating memory for tests unless the test specifically requires another allocator. The Zig test runner reports leaks from this allocator, making missing cleanup visible. Use `std.testing.io` when a test requires I/O rather than constructing a production I/O implementation unnecessarily.

If a public or internal function deliberately returns a particular error, include a test that demonstrates the error is raised under the intended condition. Every reproducible bug fix should include a regression test.

Prefer small test blocks for functions that can be checked independently. Use larger integration or end to end tests when behaviour genuinely spans several components. 

Example:

```zig
test "rejects invalid connectivity" {
    const alloc = std.testing.allocator;
    try std.testing.expectError(
        error.InvalidConnectivity,
        buildMesh(alloc, invalid_connectivity),
    );
}
```

---

## 2. Types and Data Representation

> **Principle:** Use Zig's type system to make ownership, mutability, presence, and valid states obvious.

Prefer slices for borrowed arrays:

```zig
[]const T   // read only borrowed slice
[]T         // mutable borrowed slice
```

Prefer pointers for individual borrowed objects:

```zig
*const T    // read only object
*T          // mutable object
```

Use many item and C pointers mainly at ABI/FFI boundaries. Prefer const input wherever mutation is not required as mutation should be obvious from the function signature.

Use an optional when the real question is whether a value exists:

```zig
psf: ?PSF
```

Use an enum when a value is one of several named states rather than encoding the choice as booleans:

```zig
const ThreadingMode = enum {
    single_threaded,
    multi_threaded,
};
```

Use a tagged union when a value may contain one of several distinct runtime representations:

```zig
const Shader = union(enum) {
    monochrome: ShaderMono,
    rgb: ShaderRGB,
    ir: ShaderIR,
};
```

Avoid manual tags, loosely coupled flags, and runtime type inspection when the type system can represent the state directly.

---

## 3. Variants and Dispatch

> **Principle:** Decide whether a set of implementations is open or closed, and whether the choice is known at compile time or runtime. Use the simplest dispatch mechanism that matches that decision.

For a closed finite set of runtime variants, prefer a tagged union or enum with an explicit `switch`. This keeps the supported cases visible and gives the compiler a direct dispatch structure.

For a choice known at compile time, prefer compile time type parameters, `anytype`, or other compile time dispatch. Zig's compile time duck typing is useful when several implementations provide the operations required by a kernel without needing a runtime interface.

Use function pointers or manually constructed runtime interfaces only when implementations must genuinely remain open or be selected dynamically at runtime. Avoid indirect function pointer dispatch in hot code where the implementation can be resolved earlier.

As a default:

1. use compile time dispatch when the implementation is known at compile time;
2. use tagged unions or enums when the runtime set of implementations is finite and known;
3. use function pointers only when genuinely dynamic runtime behaviour is required.

Do not create an interface abstraction for a single implementation without a real need.

---

## 4. Failure and Contracts

> **Principle:** Use errors for legitimate runtime failure, assertions for programmer errors, and `unreachable` only for states impossible by construction.

Use Zig errors for runtime failures that should be represented explicitly in the interface. Prefer explicit error sets for public APIs where they improve understanding.

```zig
const MeshError = error{
    InvalidShape,
    InvalidConnectivity,
    InvalidIndex,
};
```

Returning an error does not imply local recovery. For "run and done" tools, propagate failures directly to the application boundary when that is the clearest behaviour.

```zig
pub fn main(init: std.process.Init) !void {
    try run(init);
}
```

Use `std.debug.assert` for internal invariants and deliberate unchecked contracts:

```zig
pub fn get(self: Self, index: usize) T {
    std.debug.assert(index < self.slice.len);
    return self.slice[index];
}
```

It is acceptable for low level mathematical or performance critical APIs to define invalid indices, dimensions, or shapes as programming errors rather than ordinary runtime failures. Where a checked form is useful, provide it separately.

Use `unreachable` only when control flow has already proved that a path cannot occur:

```zig
switch (kind) {
    .scalar => handleScalar(),
    .vector => handleVector(),
    .invalid => unreachable,
}
```

Treat `catch unreachable` and `orelse unreachable` the same way. Do not use them to hide legitimate runtime failure.

Validate cheap conditions before expensive allocation or computation begins.

---

## 5. Allocation, Ownership, and Cleanup

> **Principle:** Every operation that may allocate, resize, or free memory should make the allocator explicit at the call site.

Any function that allocates memory should take an allocator parameter. Any function that takes an allocator should be assumed capable of allocation. Do not hide allocation behind mutable global state, stored allocators, or helper function calls.

Do not store an allocator in a type merely so later methods can allocate or free implicitly. A type that allocates in `init(...)` should take the allocator again in `deinit(...)`, and methods that resize or allocate should likewise take it explicitly:

```zig
var value = try Thing.init(alloc, ...);
defer value.deinit(alloc);

try value.resize(alloc, new_size);
```

This keeps allocation visible and makes it possible to judge whether a method is safe to call from a hot loop.

### Allocator naming

Use `outer_alloc` for memory that leaves the function and `local_alloc` for scratch memory whose lifetime ends with the function:

```zig
fn buildResult(
    outer_alloc: std.mem.Allocator,
    input: []const f64,
) !Result {
    var arena = std.heap.ArenaAllocator.init(outer_alloc);
    defer arena.deinit();

    const local_alloc = arena.allocator();

    const scratch = try local_alloc.alloc(f64, input.len);
    const output = try outer_alloc.alloc(f64, input.len);
    errdefer outer_alloc.free(output);

    _ = scratch;

    return .{ .values = output };
}
```

Never return pointers or slices backed by `local_alloc`.

### Process allocation

For normal applications, prefer the allocator supplied by `std.process.Init` rather than constructing an unrelated process wide allocator yourself:

```zig
pub fn main(init: std.process.Init) !void {
    const gpa = init.gpa;

    var arena = std.heap.ArenaAllocator.init(gpa);
    defer arena.deinit();

    const alloc = arena.allocator();
    try run(alloc, init.io);
}
```

`init.gpa` is Zig's default selected thread safe general purpose allocator for the target and build configuration. Debug builds arrange leak checking where possible; optimised builds select an appropriate production allocator. 

### Cleanup

Use `defer` and `errdefer` to keep cleanup next to resource acquisition. Put `errdefer` immediately after a successful allocation when later operations can fail:

```zig
const buffer = try alloc.alloc(f64, n);
errdefer alloc.free(buffer);
```

Use `defer` for unconditional cleanup at scope exit. Do not scatter cleanup across the scope when the lifetime can be expressed locally.

---

## 6. I/O and Concurrency

> **Principle:** I/O and potentially blocking or concurrent work should use the `std.Io` interface unless lower level control is genuinely required.

Functions that perform I/O should take a `std.Io` parameter explicitly. Application code should normally obtain the process I/O interface from `std.process.Init` and pass it through:

```zig
pub fn main(init: std.process.Init) !void {
    try run(init.io);
}

fn run(io: std.Io) !void {
    // I/O through the supplied interface.
}
```

Do not create a private I/O implementation deep inside normal application code merely because the function was not given one. Pass `std.Io` through the call chain when I/O is part of the operation.

Prefer `std.Io` task and synchronisation facilities for threaded or asynchronous work so the code integrates with the application's selected I/O implementation. Use `std.Thread` directly only when the `std.Io` model is unsuitable or explicit low level thread control is required.

For tests, use `std.testing.io`.

---

## 7. Compile Time Specialisation and Performance

> **Principle:** Use Zig's compile time and SIMD features when they improve generated code, not merely because they are available.

Use `comptime` when compile time knowledge improves runtime code, for example by specialising element type, fixed dimensions, node count, channel count, SIMD width, or algorithm policy. Good specialisation removes runtime dispatch or branches, enables useful unrolling or vectorisation, or otherwise improves generated code.

Avoid combinatorial `comptime` designs that generate large amounts of nearly identical code without meaningful runtime benefit. Compile time and generated code size are costs.

Use `@Vector` for explicit SIMD where the scalar implementation and data layout are already suitable for vectorisation. 

Do not use `inline` without a good reason. Use it when compile time semantics require it, when it forms part of a deliberate specialised kernel, or when measurement or generated code inspection justifies it. Use `inline for` only when compile time expansion provides a real benefit.

---

## 8. Modules and Public API Boundaries

> **Principle:** Internal code should make declaration ownership obvious. Public modules may deliberately provide a shallow and stable API.

Internal implementation code should normally import declarations from the file that defines them:

```zig
const mesh = @import("mesh.zig");
const Mesh = mesh.Mesh;
```

Public reexports are appropriate where a module deliberately acts as an API boundary or thin wrapper:

```zig
pub const Mesh = @import("mesh.zig").Mesh;
pub const Camera = @import("camera.zig").Camera;
```

A wrapper may also hide an implementation choice:

```zig
const scalar = @import("scalar.zig");
const simd = @import("simd.zig");

pub const transform = if (use_simd)
    simd.transform
else
    scalar.transform;
```

Avoid accidental reexport chains such as:

```text
root.zig -> render.zig -> geometry.zig -> mesh.zig -> Mesh
```

A declaration should normally be reexported only through the API layer that intentionally exposes it.

---

## 9. C ABI

> **Principle:** Keep the C ABI small, explicit, and built from C compatible representations. Do not expose arbitrary native Zig internals.

Use `extern struct`, C pointers, fixed primitive fields, and other C compatible types at the exported boundary. Keep native Zig containers, tagged unions, allocators, generic types, and implementation details behind that boundary.

A C facing representation can deliberately flatten richer internal types:

```zig
pub const CVec3F64 = extern struct {
    x: f64,
    y: f64,
    z: f64,
};

pub const CArray2DF64 = extern struct {
    elems: [*c]const f64,
    rows_num: usize,
    cols_num: usize,
};
```

Where the public C ABI requires a fixed precision, layout, or implementation contract, enforce that contract at compile time. Use the native Zig API for experimental configurations that should not become part of the stable C interface.

---

## 10. Expressions, Casts, and Discards

> **Principle:** Keep expressions readable enough that conversions and ignored values are deliberate and visible.

Avoid unnecessary casts. Break complicated cast chains into named intermediate values:

```zig
const raw_index: u32 = @intFromFloat(value);
const index: usize = @min(raw_index, max_index);
```

Prefer this to deeply nested builtins or function calls. Three or more nested calls are normally a sign that the expression should be split.

Use `_ = value;` deliberately. Discards should normally exist because a shared function or interface intentionally does not use a value in one implementation. If the reason is not obvious, add a short comment. Do not use discards to silence errors without understanding why the value is unused.

---

## 11. Comments and Documentation

> **Principle:** Comments should explain decisions, invariants, assumptions, and contracts that the code itself cannot make obvious.

Keep inline comments minimal. Use them to explain unusual decisions, numerical or physical assumptions, invariants, nonobvious memory layouts, ownership constraints, algorithm references, and behaviour not expressed by the type system. Do not narrate obvious code.

Public documentation should describe inputs, outputs, ownership, allocation, failure conditions, data layout, numerical assumptions, and relevant algorithms or papers where these are not obvious from the interface.

---

## 12. Formatting, Naming, and File Structure

> **Principle:** Let `zig fmt` handle mechanical style and keep the most important declarations easy to find.

Run `zig fmt` on every edited Zig file. Keep code within 100 columns where practical, use four spaces, do not use tabs, put imports at the top, and use trailing commas in multiline declarations and calls where appropriate.

Follow normal Zig naming conventions and the Performance Oriented Software Guide rule that functions are verbs and types are nouns.

For simple iteration indices, doubled lowercase names are acceptable where the meaning is obvious:

```zig
for (0..num_nodes) |nn| {
    // ...
}

for (0..num_elements) |ee| {
    // ...
}
```

Common conventions are `nn` for node, `ee` for element, `rr` for row, `cc` for column, and `ii`, `jj`, `kk` for generic indices. Use descriptive names where a short iterator would require explanation.

Single or doubled capital letters are acceptable for small obvious compile time constants:

```zig
N // count or dimension
T // type
```

Use descriptive names where a short name would be ambiguous.

A normal source file should place imports first, then public types and the external API, then implementation functions in approximate call order, then local `test` blocks at the bottom. Group related behaviour together rather than fragmenting it into unnecessary helpers.
