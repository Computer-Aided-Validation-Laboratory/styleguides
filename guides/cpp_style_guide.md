# C++ Style Guide

This guide defines the C++-specific conventions used for our scientific and engineering
software.

**Read the [Performance Oriented Software Guide](performance_oriented_software_guide.md)
first.** It defines the shared principles for correctness, Data Oriented Design, memory
behaviour, ownership, allocation, SIMD, threading, API design, numerical accuracy, and
benchmarking. This guide explains how we express those principles idiomatically in C++.

When writing C++ code our priorities are:

1. **Make it correct.**
2. **Make it fast.**
3. **Make it simple for users.**

This guide and the Performance Oriented Software Guide are authoritative for C++ code in
this project. Where neither covers a C++-specific detail, use the
[Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html) as a fallback
reference. If the Google guide or another third-party guide conflicts with any project
guide or established local convention, follow the project guidance.


## Quick Rules for New Contributors

If you are new to the project, start here:

1. Follow the Performance Oriented Software Guide.
2. Make the code correct, fast, and simple for users.
3. Design numerical code with data layout and performance in mind from the beginning.
4. There's a  `clang-format` configuration provided. Use it at your discretion.
5. Functions are verbs. Types are nouns.
6. Prefer values, RAII, and standard library types over manual resource management.
7. Use `const` wherever it communicates intent.
8. Prefer references for non-null borrowed objects.
9. Use pointers only when null or reseating is meaningful.
10. Never use raw owning pointers in normal code.
11. Avoid allocation in hot loops.
12. Keep computation separate from I/O, logging, plotting, and user interaction.
13. Fail clearly rather than continuing with questionable data.
14. Every bug fix gets a regression test.
15. Avoid unnecessary dependencies.

## Numerical Code, Performance, and Memory

Design with performance intent from the beginning, then measure and improve.

Do not deliberately write an inefficient implementation on the assumption that profiling can
fix it later. Poor data layout can make later optimisation difficult or require a complete
rewrite.

When designing numerical code, consider:

- data layout;
- array shape and axis order;
- access patterns;
- cache behaviour;
- contiguous memory access;
- allocation and copying;
- opportunities for vectorisation;
- threading and ownership;
- temporary storage.

Prefer:

- contiguous arrays;
- direct indexing;
- compact indices;
- predictable traversal;
- structure-of-arrays layouts where fields are processed independently;
- stable layouts that avoid repeated rearrangement.

Avoid:

- pointer heavy object graphs in hot paths;
- linked structures in hot paths;
- scattered heap allocations;
- repeated allocation inside loops;
- unnecessary copies;
- false sharing between threads.

A fast implementation starts with a sensible data representation.

## Public API Design

As a general guide, the public API should be easy to remember and difficult to misuse.

- Functions are actions and start with verbs.
- Types are nouns.
- Important inputs should be obvious in the function signature.
- Make ownership explicit.
- Make mutation explicit.
- Make I/O explicit.
- Use good defaults where there is one safe and unsurprising choice.
- Require explicit input when there is no safe default.
- Prefer enums and optional values over ambiguous boolean flags.
- Avoid surprising side effects.
- Keep computational code separate from I/O, logging, plotting, and user interaction.

Functions should not mutate their inputs unless mutation is the point of the operation.
Member functions may mutate `this` when their name and non-`const` signature make that
behaviour clear. Free functions that mutate arguments should say so in the name or use an
output type that makes ownership and mutation explicit.

Mutation is acceptable when it is materially better than copying, especially in performance
critical code. The important rule is that mutation must be visible to the caller, not hidden
as a side effect.

## Dependencies

Minimise dependencies. Add a dependency only when it provides substantial functionality that
would be costly, risky, or distracting to implement ourselves.

Do not add dependencies for:

- trivial utilities;
- small convenience wrappers;
- basic containers already available in the C++ standard library;
- simple maths that is already available or easy to implement clearly.

Dependencies increase:

- build complexity;
- maintenance burden;
- version compatibility work;
- platform problems.


## ABI, C APIs, and Low-Level Features

Raw pointers, raw arrays, manual memory management, unusual layout attributes, packed
structures, and similar low-level features should normally be used only when required by:

- an ABI;
- a binary file format;
- hardware;
- an external C interface;
- another explicit representation constraint.

Keep ABI-facing code isolated from normal internal data structures where practical.

For C-compatible APIs:

- wrap declarations in `extern "C"` when included from C++;
- use fixed-width C-compatible types and opaque handles rather than C++ classes, templates,
  exceptions, references, or standard-library containers;
- define ownership, buffer sizes, nullability, alignment, and error reporting explicitly;
- catch all exceptions before they reach the ABI boundary;
- use an explicit calling convention and symbol visibility policy when the platform
  requires them;
- validate C inputs before adapting them to internal C++ types.


## Formatting

- Use **clang-format** for formatting.
- Use **clang-tidy** for everyday static analysis.

- Keep code within 100 columns unless the project configuration says otherwise.
- Use blank lines to separate logical groups of statements.
- Prefer a few clear intermediate statements over one dense expression.
- Keep `if` conditions simple. Calculate complex predicates in named intermediate variables
  before the `if`.
- Keep separate statements on separate lines.

## Naming

Use descriptive names. Code should normally explain itself without requiring comments.

Use the same naming principles as the Python and Zig guides: functions are actions, types
are things, and important values have descriptive names. Prefer the project conventions
below over Google C++ naming when they differ.

### Functions

Function names use `snake_case` and should start with a **verb** that describes what the
function does. This keeps C++ APIs close to the Python API and avoids mixing Google-style
function names with the rest of the project.

```cpp
calculate_error() transform_mesh() load_image() extract_surface()
```

Avoid vague names such as `process()`, `handle()`, or `do_thing()` unless the meaning is
obvious from context.

Accessors may use noun-like names when they simply return a property:

```cpp
mesh.num_nodes() camera.position() solver.tolerance()
```

### Classes and structs

Class and struct names use `PascalCase` and should normally be **nouns**.

```cpp
Mesh Camera RasterEngine CalibrationResult
```

For related type families, put the major concept first:

```cpp
FieldScalar FieldVector FieldTensor
```

rather than:

```cpp
ScalarField VectorField TensorField
```

### Interfaces

Prefix abstract interfaces with a capital `I`:

```cpp
ISensor IRenderer IField
```

Interfaces should normally contain only pure virtual functions and a virtual destructor.

### Enumerations

Prefix enumeration classes with a capital `E`:

```cpp
EGeneratorType EInterpolationType
```

Prefer scoped enumerations:

```cpp
enum class EInterpolationType {
    nearest,
    linear,
    cubic,
};
```

### Constants

Constants use `UPPER_SNAKE_CASE`, matching the Python guide.

```cpp
constexpr double DEFAULT_TOLERANCE = 1.0e-8;
constexpr int MAX_ITERATIONS = 100;
```

Avoid magic numbers. Give important numerical values descriptive names and add a comment
when the name alone does not explain the value.

### Acronyms and abbreviations

Common, unambiguous abbreviations are fine.

```cpp
calculate_psf() load_rgb_image()
```

Use lowercase acronyms in functions and variables:

```cpp
dic_error rgb_image calculate_psf()
```

Keep acronyms uppercase in type names:

```cpp
DICErrorModel RGBImage PSFModel
```

Do not shorten names merely to save characters. Single-letter or doubled-letter names are
only acceptable for obvious indices, dimensions, or mathematical notation in a small local
scope. NumPy-style iterator names such as `ii`, `jj`, and `kk` are fine.

Use `snake_case` for variables, parameters, namespaces, and type aliases. Private class data
members have a trailing underscore:

```cpp
class Solver {
  private:
    double tolerance_;
};
```

Names beginning with an underscore or containing a double underscore are reserved in many
contexts; do not create them. Macros, when unavoidable, use `UPPER_SNAKE_CASE` with a
project-specific prefix. Name source and header files descriptively in lowercase with
underscores.

## 3. Files, Headers, and Includes

Header files define interfaces. Source files define implementations. Keep dependencies
clear and local.

- Use `.hpp` for headers and `.cpp` for source files.
- Headers must be self-contained.
- Include what you use.
- Do not rely on transitive includes.
- Keep public API declarations near the top of a header.
- Keep implementation details in source files or internal namespaces.
- Avoid putting large function bodies in headers unless they must be templates, `constexpr`,
  or performance critical inline definitions.

### Include guards

Use include guards in headers.

```cpp
#ifndef RENDER_MESH_RENDERER_H_
#define RENDER_MESH_RENDERER_H_

...

#endif // RENDER_MESH_RENDERER_H_
```

### Include order

Use this include ordering:

1. The related header.
2. C system headers.
3. C++ standard library headers.
4. Third-party headers.
5. Project headers.

Separate groups with blank lines and sort includes within each group. Prefer C++ headers
such as `<cstdint>` and `<cmath>` to their C compatibility forms.

Example:

```cpp
#include "dicsubset.hpp"

#include <cmath>
#include <vector>

#include <Eigen/Dense>

#include "dicutil.hpp"
```

### Forward declarations

Prefer including the header that defines a type when the dependency is reasonable. Use
forward declarations only when they clearly reduce dependency cost without making the code
fragile. Do not forward declare entities from another library, including the standard
library.

### Header contents and the preprocessor

A header must include everything needed to compile it in isolation. Test this directly
where practical. Put declarations in headers and definitions in source files unless a
definition must be visible for templates, is `constexpr`, or is a small performance-critical
inline function.

Avoid macros. Prefer typed constants, `constexpr` functions, templates, and enums. Never use
a macro to form a public C++ API. If a macro is required for conditional compilation,
platform integration, or testing, keep it small, give it a project-specific prefix, and do
not allow it to alter control flow unexpectedly.

Do not use C++ modules unless the project build and supported toolchains have deliberately
adopted them.

## 4. Namespaces

Use project namespaces to organise code.


- Avoid broad `using namespace` directives in source files.
- Use anonymous namespaces for source-file-local helpers.
- Keep namespace nesting shallow.
- Do not use inline namespaces unless there is a clear versioning need.
- Do not place project declarations in `std` or another library's namespace.
- Avoid mutable namespace-scope state and global singletons.
- Objects with static storage duration should normally be constant-initialised and trivially
  destructible. Prefer function-local statics when runtime initialisation is genuinely
  needed, while accounting for their hidden first-use cost.
## 5. Ownership, Lifetimes, and RAII

Make ownership obvious. Prefer values and RAII over manual resource management.

- Prefer ordinary values for objects with clear value semantics.
- Prefer standard library containers over manual memory management.
- Use `std::unique_ptr<T>` for unique ownership.
- Use `std::shared_ptr<T>` only when ownership is genuinely shared.
- Do not use raw owning pointers.
- Do not manually pair `new` and `delete` in normal code.
- Use constructors to establish invariants.
- Use destructors to release resources.
- Keep resource ownership local and visible.

Prefer:

```cpp
std::vector<double> values(num_values);
auto solver = std::make_unique<Solver>(config);
```

Avoid:

```cpp
double* values = new double[num_values];
Solver* solver = new Solver(config);
```

### Borrowing

Use references and pointers to express borrowing clearly:

- Use `const T&` for non-null read-only borrowed objects.
- Use `T&` for non-null mutable borrowed objects.
- Use `T*` when null is meaningful or when reseating is needed.
- Use a container reference only when the function specifically requires that container's
  interface, rather than merely its elements.
- A raw pointer is non-owning unless it crosses a C or hardware boundary whose contract says
  otherwise.
- A pointer, reference, or iterator into a container does not own the referenced data. Do
  not use it after the container is destroyed, moved, or modified by an operation that
  invalidates it. In particular, `std::vector` reallocation invalidates all pointers,
  references, and iterators into its elements. Erasing elements may invalidate those at or
  after the erased position. Consult the container's invalidation rules before retaining
  access across a mutation.

```cpp
double calculate_mean(std::span<const double> values) {
    double sum = 0.0;
    for (const double value : values) {
        sum += value;
    }

    return sum / static_cast<double>(values.size());
}
```

If a function mutates an argument, make that clear from the name or signature.

```cpp
void transform_mesh_in_place(Mesh& mesh, const Transform& transform);
```

### Dangling references and views

> **Principle:** Borrowed objects must not outlive the storage they refer to.

References, pointers, iterators, and non-owning views do not own their data. Do not return
them when they refer to a local variable or temporary. Do not retain them across an
operation that can invalidate them. Make the owner lifetime clear at the API boundary.

```cpp
std::string_view bad_name() {
    return std::string("temporary");  // Dangling view.
}

const Mesh& bad_mesh() {
    return Mesh{};  // Dangling reference.
}
```

Prefer returning an owning value when the result must outlive the source.

## 6. Const Correctness

Use `const` to make intent clear.

- Mark read-only parameters as `const`.
- Mark member functions as `const` when they do not mutate logical state.
- Prefer `const` local variables where it improves understanding.

Example:

```cpp
class Mesh {
  public:
    int num_nodes() const { return num_nodes_; }
    const std::vector<double>& coordinates() const { return coordinates_; }

  private:
    int num_nodes_ = 0;
    std::vector<double> coordinates_;
};
```

Do not use `const` mechanically when it makes code noisier without improving
understanding.

## 7. Containers and Data Structures

Prefer explicit, typed data structures over flexible containers.

- Prefer fixed-width integer types such as `std::int32_t` when representation or an external
  format matters.
- Prefer `std::size_t` for sizes and indices that must match standard containers, but avoid
  mixing signed and unsigned arithmetic. Convert deliberately at checked boundaries.
- Prefer `double` for general floating-point numerical work unless accuracy, storage, SIMD,
  hardware, or an external format requires another type. Avoid `long double` in portable
  interfaces.
- Prefer `std::vector<T>` for owned dynamic contiguous arrays.
- Prefer `std::array<T, N>` for fixed-size arrays.
- Prefer `std::string` for owned text.
- Prefer `std::string_view` for borrowed read-only text.
- Prefer references to standard containers for ordinary borrowed container inputs.
- Avoid raw arrays except at ABI or low-level interoperability boundaries.
- Choose containers based on access pattern, memory layout, and cache behaviour.
- Reserve capacity when the final size is known or can be estimated.

Example:

```cpp
std::vector<Node> nodes;
nodes.reserve(num_nodes);

for (int node_index = 0; node_index < num_nodes; ++node_index) {
    nodes.push_back(build_node(node_index));
}
```

### Structured return values

Prefer a named struct over a large tuple return.

```cpp
struct CalibrationResult {
    CameraIntrinsics intrinsics;
    CameraExtrinsics extrinsics;
    std::vector<double> residuals;
};
```

This is clearer than returning a tuple whose element meanings must be remembered.


## Functions and Control Flow

- Use guard clauses to reduce nesting.
- Fail early with clear error messages.
- Check everything cheap that can be checked before starting expensive work.
- Do not use errors for normal control flow.
- Do not impose arbitrary limits on function length.
- Keep related behaviour local when that makes the code easier to understand.
- Avoid single-use helper functions unless they clarify a real boundary, isolate complexity,
  or allow meaningful independent testing.
- Long argument lists are acceptable when they make important inputs explicit.
- Avoid positional booleans.

A large function is not automatically bad. A function that forces the reader to jump through
several layers of unnecessary helpers can be worse.

### Switch statements

Use a `switch` when it expresses a choice among discrete values more clearly than a chain of
conditionals.

- For a closed enum, handle every enumerator explicitly and normally omit `default`. This
  allows compiler warnings to identify switches that need updating when an enumerator is
  added.
- Use `default` when values outside the known cases are possible or when the switch is
  intentionally non-exhaustive. Handle an unexpected value explicitly rather than silently
  ignoring it.
- Do not fall through from one case to another implicitly. Use `[[fallthrough]]` when
  fallthrough is deliberate, or combine case labels when they have identical behaviour.
- End each non-falling case with `break`, `return`, or another obvious control transfer.
- Use braces around a case body when it declares variables whose scope would otherwise cross
  another case label.

```cpp
switch (interpolation_type) {
    case EInterpolationType::nearest:
        return interpolate_nearest(samples);

    case EInterpolationType::linear:
        return interpolate_linear(samples);

    case EInterpolationType::cubic:
        return interpolate_cubic(samples);
}
```

### Initialisation and conversions

- Declare variables in the narrowest practical scope and initialise them at declaration.
- Prefer brace initialisation when it prevents narrowing, but do not force one syntax where
  it reduces readability.
- Use `nullptr`, never `NULL` or `0`, for a null pointer.
- Use C++ casts. Use `static_cast` for checked-at-compile-time conversions,
  `dynamic_cast` only for a genuine polymorphic runtime check, and `const_cast` only when
  required by a correct legacy interface. Keep `reinterpret_cast` at low-level boundaries.
- Do not use C-style casts.
- Treat signed/unsigned, floating/integer, and narrowing conversions as deliberate
  operations. Check ranges when the value is not already guaranteed.
- Use `sizeof(variable)` rather than repeating its type.

### Overloads, defaults, and lambdas

Overload functions only when the variants perform the same conceptual operation and call
sites remain unambiguous. Default arguments are acceptable on non-virtual functions when
the value is stable and unsurprising; do not use them on virtual functions. Avoid trailing
return types unless the leading form is impractical.

Keep lambdas short and local. Capture explicitly when lifetime or mutation matters. Do not
use `[&]` or `[=]` when the closure may escape the current scope. Avoid default capture of
`this`; make object lifetime explicit.

Use enums, optional values, or another meaningful type when it communicates intent better
than a boolean.

Prefer:

```cpp
enum class EThreadingMode {
    single_threaded,
    multi_threaded,
};

Image render(const Scene& scene, const Camera& camera, EThreadingMode threading_mode);
```

Avoid:

```cpp
Image render(const Scene& scene, const Camera& camera, bool use_threads);
```

Likewise, prefer clear call sites:

```cpp
render(scene, camera, EThreadingMode::multi_threaded);
```

over:

```cpp
render(scene, camera, true);
```


## Errors, Assertions, and Exceptions

Prefer explicit failure over silent recovery.

Choose the project error mechanism deliberately and use it consistently. Exceptions,
and status values are acceptable only when they make failure explicit and hard to ignore. 
Try not mix error models casually across one API. Exceptions must not cross a C ABI, 
thread entry point, destructor, or other boundary that
cannot safely propagate them. Destructors and cleanup functions must not fail.

- Return explicit error values for recoverable external failures.
- Use `std::optional<T>` when the only question is whether a value exists.
- Use status or expected-style return values when an operation can fail with meaningful
  diagnostic information.
- Use assertions for programmer invariants that should never fail in valid code.
- Do not continue with questionable data when an error could change the physical or
  numerical meaning of a result.
- Treat warnings as failures in normal project development and testing.

Example:

```cpp
std::optional<int> find_node_index(const Mesh& mesh, NodeId node_id);
```

For internal invariants:

```cpp
assert(index < values.size());
```


### Undefined behaviour

Avoid out-of-bounds access, uninitialised values, signed integer overflow, invalid shifts,
use-after-free, use-after-move, invalid object lifetimes, alignment or aliasing violations,
and data races. Do not rely on behaviour that is not guaranteed by the selected C++ standard.

Validate external indices, sizes, and ranges before using them. Keep unchecked fast paths
separate from checked public interfaces, document their preconditions, and enforce those
preconditions with assertions where appropriate. Use sanitizers and compiler warnings to
find undefined behaviour during development.

```cpp
// Check before converting or indexing.
assert(index < values.size());
const double value = values[index];
```

## Classes, Abstraction, and Inheritance

Use a mixture of plain functions, structs, and classes. Do not use object oriented
programming simply because it is C++ and it's supported.

### Prefer

- plain functions;
- simple structs for data;
- classes with clear invariants;
- composition;
- dependency injection;
- abstract interfaces when an interface is genuinely useful.

### Inheritance

Inheritance is normally for a pure abstract interface.

- Avoid deep inheritance.
- Avoid mix-ins.
- Avoid multiple inheritance unless there is a strong reason and all inherited bases are
  pure interfaces.
- Prefer composition and dependency injection.
- Use `override` on every overriding function.
- Use a virtual destructor for polymorphic base classes.

Inheriting from standard library base classes whose purpose is inheritance, such as
`std::exception`, is allowed when it fits the project error model.

Example:

```cpp
class IRenderer {
  public:
    virtual ~IRenderer() = default;

    virtual Image render(const Scene& scene, const Camera& camera) const = 0;
};
```

Introduce an interface only when it solves a real problem. Prefer explicit variants and
direct dispatch when the set of behaviours is finite and known. Use a virtual interface
when implementations must remain open-ended or be selected dynamically and the indirection
does not harm a hot path.

### Structs and classes

Use a `struct` for passive data whose fields may be public and which has no invariant that
callers can break. Use a `class` when the type maintains invariants or hides representation.
Keep class data members private. Order declarations consistently, normally public API first,
then protected members if required, then private implementation.

### Constructors and special members

- Use constructors to establish valid objects.
- Mark constructors callable with one argument `explicit`, including constructors with
  additional defaulted parameters. Mark conversion operators `explicit` unless implicit
  conversion is intentionally part of the type's meaning.
- Prefer member initialiser lists and in-class member initialisers.
- Avoid doing expensive or surprising work in constructors.
- Follow the rule of zero: let value members manage resources and omit special members when
  the generated operations are correct.
- If a type owns a resource directly and needs a destructor, decide explicitly whether copy
  and move are valid; define or delete the relevant operations.
- Never call virtual functions from constructors or destructors.
- Use `= default` and `= delete` to state intent.


## Templates, `auto`, and Compile Time Code

Use templates where they reduce real duplication, express a generic algorithm clearly, or
enable important performance specialisation.

- Keep template interfaces simple.
- Prefer `concepts` when using C++20 and they make requirements clearer.
- Avoid heavy metaprogramming unless strongly justified.
- Do not use template tricks where ordinary functions or overloads are clearer.
- Avoid compile time designs that produce large amounts of code without meaningful runtime
  benefit.
- Hide metaprogramming behind a readable interface and document non-obvious constraints.
- Prefer `using` aliases over `typedef`.
- Do not specialise templates in namespace `std` unless the standard explicitly permits the
  exact specialisation.

Use `auto` when it improves readability or avoids repeating an obvious type. Do not use
`auto` when the concrete type is important context for the reader.

Prefer:

```cpp
const auto residual = measured - expected;
```

Also prefer explicit types where they clarify ownership, precision, or cost:

```cpp
std::vector<double> residuals = calculate_residuals(model, samples);
```

## Move Semantics and Copies

Be explicit about ownership transfer and copying.

- Avoid unnecessary copies of large objects.
- Use move semantics for ownership transfer.
- Do not use moved-from objects except to destroy or assign to them.
- Do not call `std::move` on local return values unless it is needed.
- Prefer passing small cheap values by value.
- Prefer passing large read-only values by `const T&`.
- Prefer returning values when ownership is transferred to the caller.

Correctness and predictability are more important than avoiding every copy.

## Allocation Strategy

Do not allocate in hot loops. This includes per:

- pixel;
- sample;
- element;
- node;
- SIMD batch;
- solver iteration.

Allocate scratch memory outside the hot path and reuse it. When the output size is known or
can be estimated, allocate once.

Example:

```cpp
std::vector<double> residuals;
residuals.resize(num_samples);

for (int sample_index = 0; sample_index < num_samples; ++sample_index) {
    residuals[sample_index] = calculate_residual(samples[sample_index], model);
}
```

## Comments and Documentation

Keep comments minimal. There is no need to narrate obvious code. The code should explain itself wherever possible.



Comments are useful for:

- explaining why a non-obvious decision was made;
- numerical or physical assumptions;
- invariants;
- array or memory layouts that are not obvious;
- ownership constraints;
- algorithm references;
- behaviour the type system does not make clear.

Do not narrate obvious code:

```cpp
// Increment index.
index++;
```

Use **Doxygen** for API documentation. Public-facing functions, classes, and interfaces should be documented where their purpose, 
parameters, assumptions, or behaviour are not immediately obvious from the interface. 
Try and add additional information rather than simply repeat the function signature. For example:

```cpp
/**
 * @brief Fills a subset of pixels from an image using centre coordinates.
 *
 * Samples pixel values from the interpolator over a grid centred at (cx, cy).
 * Equivalent to fill_from_img_subpx but takes centre coordinates rather than
 * the top-left corner, and stores both pixel coordinates and values in ss_def.
 *
 * @param ss_def     Destination subset where pixel coordinates and values are stored.
 * @param cx         X-coordinate of the CENTRE of the subset in the image.
 * @param cy         Y-coordinate of the CENTRE of the subset in the image.
 * @param interp_def Interpolator for the image from which to sample pixel data.
 */
void fill_from_centre_coords(subset::Pixels &ss_def,
                             const double cx, const double cy,
                             const Interpolator &interp_def);
```

Prefer documenting contracts, assumptions, units, ownership, and non-obvious behaviour over restating names and types already visible in the declaration.

Public documentation should be more thorough than inline comments. Document where relevant:

- inputs;
- outputs;
- ownership;
- mutation;
- allocation;
- failure conditions;
- data layout;
- numerical assumptions;
- algorithm or paper references.

## Testing and Correctness

Testing is part of the implementation, not an optional extra. Where possible, test against:

- analytic solutions;
- invariants;
- conservation laws;
- independently calculated values;
- manufactured solutions;
- known limiting behaviour.

### Use gold regression tests

Gold regression tests are strongly encouraged for end-to-end numerical workflows.

A gold file should represent a known correct result that does not change merely because the
API or implementation was refactored.

For performance code, verify:

- scalar correctness;
- SIMD equivalence where required;
- single-threaded and multi-threaded equivalence where required;
- different supported precisions where relevant;
- representative data layouts and workloads.

Approximate maths, lookup tables, lower precision, reduced sampling, or other performance
approximations require an explicit accuracy contract and verification.

Avoid tests that only duplicate existing coverage, assert that removed functionality is
absent, or reach deep into implementation details without a behavioural reason. Tests should
protect useful behaviour while leaving room to refactor the implementation.