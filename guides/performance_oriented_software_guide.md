# Performance Oriented Software Guide

This guide defines how we design and implement performance oriented software for
scientific and engineering applications.
It is targeted at compiled languages such as C, C++,
and Zig,
although many of the principles apply more broadly.
Language specific guides define how these principles are expressed idiomatically in
each language.

Our priorities are:

1. **Make it correct.**
2. **Make it fast.**
3. **Make it simple for users.**

These priorities must be considered together during design.
Do not deliberately build software in a form where performance or
usability will later require a complete rewrite.
This guide is intentionally prescriptive.
Rules exist because they support correctness, performance, simplicity,
or some combination of the three.
Where a rule clearly harms those goals in a particular case,
use engineering judgement and document the reason.

---

## Quick Rules for New Contributors

- verify correctness with tests and independent reference calculations;
- every reproducible bug fix should have a regression test;
- design for performance from the beginning;
- design around the data and how it moves through memory;
- prefer contiguous memory and predictable traversal;
- avoid pointer chasing and object graphs in hot code;
- prefer explicit variants and direct dispatch over inheritance and
  virtual dispatch;
- functions are verbs and types are nouns;
- make ownership, mutation, allocation, I/O,
  and failure behaviour explicit;
- do not allocate in hot loops;
- fail early before expensive work begins;
- use assertions for programmer errors and broken invariants;
- keep hot paths small and predictable;
- get scalar code right first while designing work to
  be independently partitioned;
- then vectorise with Single Instruction Multiple Data (SIMD);
- then multithread;
- design threads around ownership, not mutexes,
  where possible;
- establish numerical equivalence between scalar, SIMD,
  and threaded implementations where appropriate;
- benchmark approximations against a direct or higher fidelity implementation;
- A/B test meaningful optimisations on representative workloads;
- minimise dependencies;
- keep public APIs simple;
- prefer API stability,
  but do not preserve a bad API merely to avoid a breaking change.

---

## 1. Correctness and Verification

> **Principle:** Performance is irrelevant if the result is wrong.
> Build independent evidence that the implementation is correct.

Every reproducible bug fix should include a regression test.
Prefer tests based on analytic results, independently calculated values, invariants,
independent oracle implementations, carefully chosen golden results,
and end to end behaviour.
Keep separate reference implementations simple even when they are slower.

Performance optimisations require particularly strong verification because
they often introduce duplicated implementations, reordered floating point operations,
approximate calculations,
or more complex control flow.
Verify scalar correctness first,
then establish equivalence for SIMD, threaded, precision, layout,
and execution variants where relevant.
Where bitwise equality is practical, require it.
Where floating point ordering makes exact equality unreasonable,
define and test an appropriate numerical tolerance.

A faster approximation is acceptable only when its accuracy is known and
verified.
Lookup tables, reduced precision, reduced sampling, approximate maths, interpolation,
truncated series, relaxed solver tolerances,
and similar techniques must be compared against a direct or
higher fidelity implementation.
Benchmark both runtime and numerical error.
An optimisation that violates the required accuracy is incorrect.

If nondeterminism is intentional, document why and
define the acceptable numerical behaviour.

---

## 2. Design for Performance from the Start

> **Principle:** Performance is an architectural property.
> A profiler can find bottlenecks,
> but it cannot repair a design that moves data poorly, allocates constantly,
> or requires unnecessary synchronisation.

Do not interpret "profile before optimising" as permission to
ignore obvious performance constraints during design.
Before implementing a substantial computational feature, consider algorithmic complexity,
data layout, memory traversal, allocation, temporary storage, working set size,
SIMD suitability, opportunities for independent work, thread ownership,
and likely I/O costs.

The normal development process is to choose an algorithm and layout that
are plausibly efficient, implement a correct scalar version,
benchmark representative workloads, improve the hot paths,
and A/B test meaningful optimisations.
Keep an optimisation only when it improves the relevant metric without
violating correctness or making the design unnecessarily complex.

Many plausible optimisations are neutral or harmful.
Do not assume an optimisation works because it sounds lower level or
more clever.

---

## 3. Data and Memory for Performance

> **Principle:** Performance comes from moving the right data through memory
> efficiently.
> Design data structures for the dominant computation,
> not for a conceptual object hierarchy.

### Memory hierarchy

A useful simplified model is:

```text
registers                 tiny, fastest
    ↓
L1 cache                  tens of KB per core
    ↓
L2 cache                  roughly hundreds of KB to a few MB per core
    ↓
L3 / last level cache     typically many MB, usually shared
    ↓
main memory               GBs+, much slower
```

Exact sizes and latencies vary by processor,
but the design lesson is consistent:
reuse data while it is close to the processor and
avoid moving unnecessary bytes.
A working set that fits in L1 or L2 can behave very differently from one that
repeatedly falls through to lower cache levels or RAM.

### Data Oriented Design

Data Oriented Design is the default performance philosophy for
our computational code.
Prefer contiguous arrays, direct indexing, predictable traversal, compact representations,
independent chunks of work,
and layouts that naturally map onto the calculation.
Avoid pointer heavy object graphs, scattered heap allocations,
linked structures in hot paths, repeated pointer chasing,
and behavioural decomposition that obscures data flow.

When a finite set of behaviours is known, prefer explicit data plus direct
dispatch over an inheritance hierarchy whose main purpose is to hide which
implementation is running.
In languages that support object oriented runtime polymorphism,
do not use inheritance and virtual dispatch as the default architecture for
numerical kernels.

### Structure of Arrays and Array of Structures

Choose the layout that matches how fields are consumed.
If an algorithm repeatedly processes one field across many items,
Structure of Arrays often improves contiguous access and SIMD:

```text
x[]
y[]
z[]
```

If the algorithm normally consumes all fields of one item together,
Array of Structures may be more appropriate:

```text
{x, y, z}
{x, y, z}
{x, y, z}
```

Do not apply Structure of Arrays mechanically.
Match the layout to the real access pattern.

### Cache lines and predictable access

Memory moves through the cache hierarchy in cache line sized blocks,
typically around **64 bytes** on modern desktop and server CPUs.
Loading one value therefore usually brings nearby values with it.
Sequential and regular access patterns make useful use of each cache line and
are easier for hardware prefetchers to predict.

Prefer sequential traversal, contiguous arrays, simple strides, sequential writes,
small working sets,
and reuse of values while they are hot.
Avoid random pointer chasing, unpredictable gathers, repeated transposes,
unnecessary copies,
and touching only a few bytes from many different cache lines.

Modern CPUs use hardware prefetchers to recognise regular access patterns and
fetch upcoming cache lines before they are needed.
Predictable traversal can therefore be substantially faster than irregular
traversal even when the number of logical loads is similar.

Cache coherence also operates at the cache line level.
Two threads writing different values that occupy the same cache line can
repeatedly invalidate each other's copies even though they never write the same
variable.
This is **false sharing** and should be avoided.

For memory bound work, moving fewer useful bytes is often more important than
saving a small number of arithmetic instructions.
In many cases storing a pre-computed value from a set of
small arithmetic operations will be more computationally expensive than recomputing that
value on the fly and reducing the amount of data passing through the cache
hierarchy.

---

## 4. Ownership, Mutation, and Allocation

> **Principle:** Ownership, mutation,
> and allocation must be visible.
> A caller should be able to determine whether an operation is safe to place in
> a hot loop.

For returned pointers, buffers, containers,
and objects that own memory, make the ownership contract clear.
Prefer borrowed immutable input where practical, explicit mutable output,
returned owning values where transfer is clear,
local scratch storage with a bounded lifetime,
and deterministic cleanup.
Avoid hidden ownership transfer, ambiguous shared mutable ownership,
mutable global state,
and references to temporary storage.

Mutation should be deliberate and obvious from the interface.
Prefer explicit output buffers or returned values over hidden mutation of
unrelated state.

Allocation is both a performance event and an ownership event.
APIs that may allocate should make that fact apparent through the language's
normal conventions.
Do not hide allocation behind global state, unrelated objects, innocent looking accessors,
helper calls whose allocating behaviour is unclear,
or repeated dynamic growth.

Do not allocate per sample, SIMD batch, solver iteration,
or other inner work item.
Allocate scratch memory outside the hot path and reuse it.
Prefer reusable buffers, caller provided output buffers for frequently called kernels,
presized containers,
and predictable lifetime regions.

When output size is not known in advance, consider multiple passes:

1. count;
2. allocate once;
3. populate.

Multiple linear passes over contiguous data are often cheaper and
simpler than repeated dynamic reallocation.

---

## 5. Execution Strategy

> **Principle:** Build the correct scalar architecture first,
> but design the scalar work so that SIMD and threading do not require a redesign.

The normal development order is:

1. scalar, single threaded;
2. SIMD, single threaded;
3. multithreaded.

While writing the scalar implementation, deliberately structure work into
independent tiles, blocks, rows, elements, batches,
or regions where the algorithm allows it.
Ask whether each chunk can own its output, whether scratch storage can be local to
the chunk or worker, whether reductions can happen after independent work,
and whether the data layout will vectorise cleanly.

### Single Instruction Multiple Data (SIMD)

SIMD works best when several independent pieces of
work consume regular contiguous data.
Prefer lane independent operations, contiguous loads and writes, masks or isolated tails,
hoisted invariants,
and small local accumulation.
Avoid repeated gathers, unnecessary horizontal reductions,
per lane branching where layout can remove it,
and vectorising a poor memory access pattern.

Choose the SIMD direction based on data movement,
not merely on which loop looks easiest to vectorise.
Keep a scalar reference implementation where practical.

### Multithreading

Parallelism should come from independent ownership,
not from synchronising shared mutable state:

```text
worker 0 owns region 0
worker 1 owns region 1
worker 2 owns region 2
worker 3 owns region 3
```

Prefer independent work units, exclusive output ownership, per thread scratch,
tasks coarse enough to make scheduling cheap relative to useful work,
and merge or reduction phases after independent work.
Avoid mutexes in hot paths, atomics where ownership can remove the need for them,
fine grained scheduling without evidence it helps, shared mutable structures,
and false sharing.

Parallelism should be an architectural property,
not something bolted onto globally shared scalar code afterwards.

---

## 6. Dispatch and Specialisation

> **Principle:** Resolve choices as early as practical.
> Use static or compile time dispatch for known implementations and
> runtime dispatch only when behaviour is genuinely dynamic.

When the set of behaviours is finite and known, prefer enums, tagged variants,
direct switches,
or compile time policies.
A direct switch over explicit states is often easier to understand, optimise,
and test than an inheritance hierarchy or manual runtime interface.

Function pointers are appropriate when behaviour must genuinely be selected at
runtime.
Avoid function pointer dispatch in hot inner loops where
the implementation is known at compile time.
Indirect calls can inhibit inlining and other compiler optimisations and can be harder for
the processor to predict.

Prefer, in order:

1. compile time or static dispatch where the implementation is known;
2. an enum or tagged variant with explicit dispatch where
   the runtime choices are finite;
3. function pointers or other runtime interfaces where
   genuinely dynamic behaviour is required.

Compile time specialisation is useful when fixed types, dimensions, SIMD widths,
algorithm policies,
or other known values materially improve generated code.
Use it to remove runtime dispatch or branches,
enable vectorisation or unrolling,
or improve layout.
Avoid combinatorial specialisation that produces large amounts of
nearly identical code without meaningful runtime benefit.
Compile time and generated code size are resources too.

---

## 7. Failure and API Design

> **Principle:** Fail early on legitimate runtime failure,
> assert programmer errors,
> and make ownership, mutation, allocation, I/O,
> and failure behaviour obvious at the API boundary.

Use explicit runtime failure for invalid external input, environmental failure,
unsupported input, allocation failure,
and other legitimate runtime outcomes.
Use assertions for deliberate unchecked contracts and
broken internal invariants.
Reserve language specific unreachable or assume mechanisms for states that
are genuinely impossible by construction.

In "run and done" scientific software, validate everything practical before
expensive allocation or computation begins.
Cheap checks on dimensions, shapes, indices, metadata, configuration, supported types,
resources,
and obvious numerical constraints should happen before large allocations,
long computations,
or thread creation.
Failing after minutes of work for an error that
could have been detected immediately is both a correctness and
usability failure.

Public computational APIs should make important inputs obvious and
should expose meaningful side effects.
A caller should be able to determine whether an operation allocates, mutates, performs I/O,
can fail,
or transfers ownership.
Use good defaults where one safe and unsurprising choice exists.
Require explicit configuration when it does not.
Avoid boolean heavy APIs when the real state is one of several named modes.

Keep computation separate from file I/O, plotting, logging, status reporting,
and user interaction wherever practical.
Numerical kernels should normally be silent.

API stability is valuable,
but do not preserve a poor design merely to avoid a breaking change.
This is especially important in new projects where
the API is still being discovered.

---

## 8. Benchmarking and Performance Validation

> **Principle:** Measure meaningful optimisations on representative workloads.

Strongly prefer A/B testing of performance changes.
Use workloads that reflect real use rather than tiny synthetic cases that
accidentally favour one implementation.

Measure the quantity that matters,
which may include elapsed time, throughput, latency, allocation count, memory use,
scaling with thread count, compile time,
or binary size.
Record enough information to reproduce important performance conclusions.

Treat optimisations that produce no meaningful gain as suspect,
particularly when they make the code harder to understand.
A profiler should refine an architecture that was designed sensibly;
it should not be expected to rescue an architecture that
fundamentally moves data poorly.

---

## 9. Boundaries and Dependencies

> **Principle:** Keep external representation constraints at the boundary and
> minimise dependencies that increase the long term cost of the software.

ABIs, binary file formats, hardware interfaces,
and external libraries may require packed structures, unusual integer widths,
alignment constraints, sentinel representations,
or other specialised layouts.
Keep these representations at the boundary unless the same layout is also appropriate for
the internal computation.
Convert once at the boundary when doing so gives the computational core a better
representation.

Dependencies increase build complexity, compile time, maintenance burden,
version compatibility work, packaging complexity and distribution.
Add one when it provides substantial functionality that would be costly, risky,
technically specialised,
or unreasonable to maintain ourselves.
Do not add dependencies for trivial utilities,
basic containers already provided by the language,
or small functionality that can be implemented clearly and safely.

---

## 10. General Code Conventions

> **Principle:** Code should make its role obvious without requiring the reader to
> inspect unnecessary implementation detail.

Functions are verbs because they perform actions:

```text
calculateError()
transformMesh()
loadImage()
extractSurface()
```

Types and structs are nouns because they represent data and/or things:

```text
Mesh
Camera
RasterEngine
CalibrationResult
```

Prefer direct control flow.
Use guard clauses to reduce nesting, break complicated expressions into named
intermediate values,
and avoid unnecessary layers of tiny helpers that force the reader to jump around without
improving reuse or clarity.
A large function is not automatically bad;
fragmented logic can be worse.

Comments should explain why an unusual decision exists, numerical or physical assumptions,
invariants, ownership constraints, nonobvious data layouts,
algorithm references,
and behaviour the type system cannot express.
Do not narrate obvious code.
