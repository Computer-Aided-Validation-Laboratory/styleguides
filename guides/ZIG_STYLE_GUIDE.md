# Zig Style Guide

These are our rules for Zig code written for scientific and engineering software. Our goal is 
not to use every feature Zig provides. Our goal is to write code that is correct, fast, 
explicit, predictable, and easy for other engineers to understand.

When writing Zig code our priorities are:

1. **Make it correct.**
2. **Make it fast.**
3. **Make it simple for users.**

These three priorities should be considered together while designing the software such that the project does not end up in a state where 2. or 3. require a complete re-write from the start.

---

## Quick Rules for New Contributors

If you are new to the project, start here:

- make ownership explicit;
- use the type system;
- prefer enums, optionals, and tagged unions over flags;
- fail early;
- assert internal invariants;
- use slices;
- avoid allocation in hot loops;
- think about the cache;
- prefer SoA where the access pattern benefits;
- get scalar code right first;
- then use `@Vector`;
- then multithread;
- design threads around ownership, not mutexes;
- use `comptime` for runtime performance, but remember that compile time is a cost;
- minimise dependencies;
- keep the API simple.

---

## 1. Core Principles

- Prefer explicit code over clever code.
- Design around the data and how it moves through memory.
- Make ownership, allocation, I/O, and failure behaviour obvious.
- Fail early when inputs are invalid.
- Keep hot paths small and predictable.
- Design work to be independent so that parallelism does not require locks.
- Keep the public API simple even when the implementation is highly specialised.
- Prefer simple language features and direct control flow.
- Do not introduce abstractions until they solve a real problem.
- Keep behaviour local when splitting it apart would make the implementation harder to
  understand.

---

## 2. Formatting and File Structure

Use normal Zig style and let `zig fmt` do the mechanical formatting.

- Run `zig fmt` on every edited Zig file.
- Keep code within 100 columns.
- Use four spaces for indentation.
- Do not use tabs.
- Put imports at the top of the file.
- Put public types and the external API near the top.
- Group functions by functionality and approximate call order.
- Put private helpers below the public or higher level functions that call them.
- Use trailing commas in multiline calls, declarations, and literals if the line is long.
- Keep separate statements on separate lines.

Do not fight `zig fmt`. Structure the code so that the formatted result is readable.

---

## 3. Naming

Use normal Zig casing conventions.

### Functions are verbs

Function names should describe an action.

```zig
calculateError()
transformMesh()
loadImage()
extractSurface()
```

Avoid noun like function names that hide what the function actually does.

### Structs and types are nouns

```zig
Mesh
Camera
RasterEngine
CalibrationResult
```

A type represents a thing. A function does something.

### Iterator names

Use doubled lowercase letters for simple iteration indices.

```zig
for (0..num_nodes) |nn| {
    // ...
}

for (0..num_elements) |ee| {
    // ...
}
```

Common conventions are:

- `nn` - node
- `ee` - element
- `rr` - row
- `cc` - column
- `ii`, `jj`, `kk` - generic dimensions or indices

Use descriptive names when the meaning of a short iterator is not obvious.

### Small comptime constants

Single or doubled capital letters are acceptable for small and obvious comptime
constants.

```zig
N
D
NN
```

Use descriptive names when a short name would require explanation.

### Abbreviations

Abbreviations are fine when they are common and unambiguous.

Prefer:

```zig
calc_error
```

only where Zig naming conventions or surrounding code make that form appropriate.
Do not shorten names merely to save characters.

---

## 4. Types and Intent

Use Zig's type system to make intent obvious.

- Prefer explicit types where they improve understanding.
- Prefer enums over booleans for configuration choices.
- Prefer optionals when the real question is whether a value exists.
- Prefer tagged unions when a value may be one of several distinct cases.
- Prefer slices for arrays of borrowed data.
- Prefer `[]const T` for read-only borrowed data.
- Prefer `[]T` for mutable buffers.
- Prefer `*const T` or `*T` for individual objects.
- Use many item pointers mainly at ABI or FFI boundaries.
- Avoid unnecessary casts.
- Break complicated cast chains into named intermediate values.

Prefer:

```zig
const raw_index = @intFromFloat(value);
const bounded_index = @min(raw_index, max_index);
const index: usize = @intCast(bounded_index);
```

Avoid:

```zig
const index: usize = @intCast(@min(@intFromFloat(value), max_index));
```

Three or more nested function calls or builtins are normally a sign that the
expression should be split.

---

## 5. Enums, Optionals, and Tagged Unions

Avoid boolean heavy APIs. A boolean is appropriate when the meaning really is an obvious 
yes/no state:

```zig
if (is_visible) {
    // ...
}
```

For configuration choices, prefer an enum:

```zig
const ThreadingMode = enum {
    single_threaded,
    multi_threaded,
};
```

For presence or absence, prefer an optional:

```zig
psf: ?PSF
```

For mutually exclusive data variants, prefer a tagged union:

```zig
const Shader = union(enum) {
    monochrome: ShaderMono,
    rgb: ShaderRGB,
    ir: ShaderIR,
};
```

Tagged unions are preferred to loosely coupled flags, manual tags, runtime type
inspection, or class like polymorphism.

---

## 6. Functions and Control Flow

- Use guard clauses to reduce nesting.
- Fail early.
- Validate cheap conditions before expensive work begins.
- Prefer `for` loops when iterating over a range, slice, array, or collection.
- Use `while` loops for convergence loops, retries, irregular termination, or cases where
  `while` better represents the algorithm.
- Do not impose arbitrary limits on function length.
- Keep related behaviour together when locality makes the implementation clearer.
- Avoid deeply nested control flow.
- Break complicated predicates into clearly named intermediate values.
- Do not use errors as routine control flow.

A large function is not automatically bad. A function that forces the reader to jump
through several layers of unnecessary helpers can be worse.

---

## 7. Errors, Assertions, and Failure

Fail loudly rather than attempting to continue with invalid state.

### Input validation

Invalid user or API input should return an explicit error. Prefer explicit error sets for 
public APIs.

```zig
const MeshError = error{
    InvalidShape,
    InvalidConnectivity,
    InvalidIndex,
};
```

Check everything practical before starting expensive allocation or computation.

### Internal invariants

Use `std.debug.assert` for internal conditions that should always be true when the
program is correct. This is particularly useful in performance-critical code:

```zig
std.debug.assert(index < values.len);
```

Assertions provide strong checking in Debug builds and compile away in ReleaseFast. Use the 
distinction:

- **Invalid input or recoverable external failure -> error**
- **Broken internal invariant on the hot path -> assertion**

### Crashes

`unreachable`, `catch unreachable`, and `orelse unreachable` represent a crash. Use them only 
when failure genuinely means the program is internally inconsistent and continuing would be 
wrong. Do not use them to suppress an error that should be handled or returned.

### Cleanup

Use `defer` and `errdefer` aggressively to make cleanup local and reliable.

---

## 8. Memory Ownership and Allocators

Allocation must be explicit. Any function that allocates memory should take an allocator 
parameter. Any function that takes an allocator is assumed to allocate. Do not hide allocation 
behind:

- global state;
- unrelated objects;
- implicit allocators;
- unexpected helper calls.

### Allocator naming

Name the allocator passed into an allocating function:

```zig
outer_alloc
```

If a function uses temporary scratch allocation, create a function local arena and call
its allocator:

```zig
local_alloc
```

The rule is:

> **`outer_alloc` is for memory that leaves the function.**
>
> **`local_alloc` is for scratch memory that dies with the function.**

Example:

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

Never return a pointer or slice backed by `local_alloc`.

### Ownership

- The allocator that creates returned memory is responsible for the ownership contract.
- Document ownership for returned slices, pointers, and structs containing allocations.
- Use `deinit` for types that own multiple allocations or require structured cleanup.
- Use `errdefer` immediately after successful allocations when later operations may fail.
- Do not return partially initialised owning values without a clear cleanup path.
- Store an allocator in a long lived type only when that type owns memory that needs it
  later for cleanup or resizing.
- Do not use mutable global allocator state.

---

## 9. Allocation Strategy

Do not allocate in hot loops. This includes per:

- pixel;
- sample;
- element;
- node;
- SIMD batch;
- solver iteration.

Allocate scratch memory outside the hot path and reuse it. When the output size is not known 
in advance, prefer multiple passes:

1. count;
2. allocate once;
3. populate.

Multiple linear passes over contiguous data are often preferable to repeated dynamic
allocation or container growth. Also:

- pre-size dynamic containers where possible;
- reuse scratch buffers;
- avoid repeated append/reallocation patterns;
- prefer caller provided output buffers for frequently called kernels where appropriate;
- treat unexpected allocation in a hot path as a performance bug.

---

## 10. Data Oriented Design

Design data structures around how the computation consumes the data.

Prefer:

- contiguous arrays;
- direct indexing;
- compact indices;
- predictable traversal;
- structure-of-arrays layouts where fields are processed independently;
- data layouts that map naturally to SIMD;
- stable layouts that avoid repeated rearrangement.

Avoid:

- pointer heavy object graphs;
- linked structures in hot paths;
- scattered heap allocations;
- pointer chasing;
- repeated gathers from unrelated memory locations.

### Structure of Arrays

Prefer Structure of Arrays when it improves contiguous access, cache behaviour, or SIMD.

For example, prefer thinking about:

```text
x[]
y[]
z[]
```

rather than automatically storing:

```text
{x, y, z}[]
```

when the algorithm processes `x`, `y`, and `z` independently. Do not apply SoA mechanically. 
Choose the layout that matches the actual access pattern.

---

## 11. Cache and Memory Traffic

Treat cache behaviour as a primary design constraint.

Consider from the beginning:

- traversal order;
- data layout;
- working set size;
- stride;
- allocation pattern;
- temporary buffers;
- read/write traffic;
- reuse distance.

Prefer:

- contiguous access;
- sequential writes;
- incrementing indices;
- small local working sets;
- loading data once and reusing it;
- hoisting loop invariants;
- compact per-thread scratch buffers.

Avoid:

- reconstructing array indices repeatedly in the innermost loop;
- unnecessary copies;
- repeated transposes or rearrangements;
- read-modify-write traffic where a sequential write is possible;
- false sharing between threads.

For memory bound work, moving fewer bytes is often more important than reducing a small
amount of arithmetic.

---

## 12. Performance Development Order

Performance work should normally proceed in this order.

### Stage 1 - Fast scalar, single threaded

First make the scalar implementation:

- correct;
- algorithmically sound;
- cache friendly;
- allocation aware;
- data oriented;
- easy to verify.

Do not begin with threading or SIMD around a poor data layout.

### Stage 2 - SIMD, single threaded

Once the scalar structure is sound, vectorise the hot work using `@Vector`. Prefer lane 
independent work and contiguous data.

### Stage 3 - Multithread

Only after the single-threaded implementation is good should the work be split across
threads. This order keeps performance work understandable and makes it much easier to identify
where speedup actually comes from.

---

## 13. SIMD

Use `@Vector` for explicit SIMD in performance critical kernels.

- Prefer SIMD across independent pieces of work.
- Keep lanes full where possible.
- Use masks or isolated tails for incomplete vectors.
- Prefer contiguous loads.
- Avoid designs that require repeated gathers.
- Avoid horizontal reductions in the hot path where the algorithm allows.
- Hoist invariant values before vector loops.
- Accumulate in registers or compact local buffers before writing to memory.
- Keep a scalar reference or correctness path where practical.

Choose SIMD direction based on data movement, not merely on which loop looks easiest to
vectorise.

---

## 14. Multithreading and Synchronisation

Design the architecture so that threads do not need to coordinate.

Prefer:

```text
worker 0 owns region 0
worker 1 owns region 1
worker 2 owns region 2
worker 3 owns region 3
```

over several workers mutating the same data.

### Prefer

- independent work units;
- exclusive output ownership;
- coarse enough tasks to amortise scheduling;
- per-thread scratch storage;
- merge/reduction phases after independent work.

### Avoid

- mutexes in hot paths;
- shared mutable state;
- atomics where ownership can remove the need for them;
- fine grained task scheduling;
- false sharing;
- several threads repeatedly writing to nearby or shared cache lines.

Parallelism should be an architectural property, not something bolted onto shared state
code afterwards.

---

## 15. `comptime`

`comptime` is one of Zig's strongest tools for runtime performance, but compile time is
also a resource. Use `comptime` where it:

- removes runtime dispatch;
- specialises hot kernels;
- makes dimensions known;
- enables unrolling or vectorisation;
- removes runtime branching;
- improves generated code.

Examples of useful compile time specialisation include:

- element type;
- node count;
- channel count;
- SIMD width;
- algorithm policy.

But every additional specialisation has a compilation cost. Avoid combinatorial `comptime` 
designs where many independent parameters produce large numbers of nearly identical generated 
functions without meaningful runtime benefit.

> **Use `comptime` deliberately. Runtime speed and compile time are a tradeoff.**

---

## 16. `inline` and `inline for`

Do not sprinkle `inline` throughout the code in the hope that it will make things faster.

Use `inline` when:

- compile time semantics require it;
- the operation is part of a specialised hot kernel;
- measurement or generated code inspection justifies it.

Use `inline for` when the iteration domain is genuinely compile time and expanding the
loop enables useful specialisation. Do not use `inline for` merely because the loop bound 
happens to be known.

---

## 17. Compile Time Dispatch and Interfaces

Prefer compile time dispatch for performance critical code.

Zig's compile time generic and duck typing model is useful when several implementations
share the operations required by a kernel.

Prefer:

- compile-time type parameters;
- duck typing where the required interface remains clear;
- tagged unions where runtime variants are known and finite;
- `switch` on tagged unions for explicit runtime dispatch.

Avoid runtime function pointer dispatch in hot code where compile time dispatch can
remove it. Use function pointers when runtime polymorphism is genuinely required. Do not 
introduce an interface abstraction for a single implementation without a real
need.

---

## 18. API Design

The public API should be simple and predictable.

- Functions are actions and start with verbs.
- Types are nouns.
- Important inputs should be obvious in the function signature.
- Make allocation explicit.
- Make I/O explicit.
- Use good defaults where there is one safe and unsurprising choice.
- Require explicit input when there is no safe default.
- Prefer enums and optionals over ambiguous boolean flags.
- Avoid surprising side effects.
- Keep computational code separate from I/O, plotting, logging, and user interaction.

A caller should be able to understand from the function signature whether the function:

- allocates;
- mutates output;
- performs I/O;
- can fail.

---

## 19. Mutation

Prefer immutable borrowed input where practical.

Use:

```zig
[]const T
```

for input data that should not be modified. Use mutable slices only when mutation is 
intentional and obvious. Prefer output buffers or returned owned values over hidden mutation.
If an API mutates an input object in a way that might surprise the caller, make that
behaviour explicit in the function name or signature.

---

## 20. I/O and Logging

Computation should normally be silent. Do not put printing or logging into numerical kernels. Keep:

- computation;
- file I/O;
- status reporting;
- debugging output;

separate wherever practical.

For long running operations, status output may be useful, but it should be possible to
disable it. Debugging output should:

- target one or two representative cases;
- avoid printing per pixel, node, element, sample, or solver iteration;
- be removed once the problem is resolved and or use `comptime` and a build configuration option to remove it.

---

## 21. Dependencies

Minimise dependencies. Zig makes it practical to build substantial software with the standard 
library with project specific code. Add a dependency only when it provides substantial 
functionality that would be costly, risky, or unreasonable to maintain ourselves.

Do not add dependencies for:

- trivial utilities;
- small convenience wrappers;
- basic containers;
- simple maths already available in Zig or easy to implement clearly.

Dependencies increase:

- build complexity;
- compile time;
- maintenance burden;
- version compatibility work;
- platform problems.

---

## 22. Imports and Re-Exports

Import declarations from the file that defines them. Do not re-export imported declarations.

Avoid:

```zig
pub const Mesh = @import("mesh.zig").Mesh;
```

Prefer:

```zig
const mesh = @import("mesh.zig");
const Mesh = mesh.Mesh;
```

where the type is needed.

Do not build public namespace layers by repeatedly re-exporting declarations through
other files. Keep ownership of a declaration obvious by importing it from its source.

---

## 23. ABI and FFI Features

Sentinel pointers, many-item pointers, packed structs, bitfields, unusual integer widths,
and similar low-level representations should normally be used only when required by:

- an ABI;
- a binary file format;
- hardware;
- an external C interface;
- another explicit representation constraint.

Do not use specialised representation features merely because they exist. Keep ABI-facing code 
isolated from normal internal data structures where practical.

---

## 24. Comments and Documentation

Keep comments minimal. The code should explain itself wherever possible. Use comments to 
explain:

- why an unusual decision exists;
- numerical or physical assumptions;
- invariants;
- array or memory layouts that are not obvious;
- ownership constraints;
- algorithm references;
- behaviour the type system does not make clear.

Do not narrate obvious code. Public documentation should be more thorough than inline 
comments. Document where relevant:

- inputs;
- outputs;
- ownership;
- allocation;
- failure conditions;
- data layout;
- numerical assumptions;
- algorithm or paper references.

---

## 25. Testing and Correctness

Every reproducible bug fix should include a regression test. Use a balance of:

- analytic tests;
- independently calculated values;
- invariants;
- gold regression tests;
- end-to-end tests.

Prefer tests that remain valid when implementation details or APIs are refactored. For 
performance code, verify:

- scalar correctness;
- SIMD equivalence;
- single-threaded and multi-threaded equivalence where required;
- different supported precisions where relevant;
- representative data layouts and workloads.

Approximate maths, LUTs, lower precision, reduced sampling, or other performance
approximations require an explicit accuracy contract and verification.

---

## 26. Performance Mindset

Do not write obviously inefficient code and assume profiling will rescue it later. Think about 
performance while designing:

- data layout;
- access pattern;
- cache behaviour;
- allocation;
- temporary storage;
- SIMD suitability;
- opportunities for independent work;
- thread ownership.

Then benchmark and profile the implementation to find the remaining bottlenecks. A profiler 
can identify where time is being spent. It cannot cheaply repair an architecture that 
fundamentally moves data poorly or requires unnecessary synchronisation.

---
