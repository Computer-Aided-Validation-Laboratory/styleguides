# Python Style Guide

This guide defines the Python style used for our scientific and
engineering software.
It is deliberately prescriptive.
Python has many ways to solve the same problem;
we use a smaller, explicit subset that keeps code readable, fast,
predictable and approachable for our users.

When writing Python code our priorities are:

1. **Make it correct.**
2. **Make it fast.**
3. **Make the user interface simple and intuitive.**

Prefer explicit, readable code over clever Python.
Design numerical code with data layout, memory access and performance in mind from
the beginning.
Public interfaces should be easy to remember,
difficult to misuse and unsurprising.

## Table of Contents

- [Quick Rules for New Contributors](#quick-rules-for-new-contributors)
- [1. Formatting](#1-formatting)
- [2. Naming](#2-naming)
- [3. Type Hints](#3-type-hints)
- [4. Functions and Control Flow](#4-functions-and-control-flow)
- [5. Errors, Warnings and Validation](#5-errors-warnings-and-validation)
- [6. Data Structures](#6-data-structures)
- [7. Classes, Abstraction and Inheritance](#7-classes-abstraction-and-inheritance)
- [8. Restricted and Banned Python Features](#8-restricted-and-banned-python-features)
- [9. Argument Mutation and Side Effects](#9-argument-mutation-and-side-effects)
- [10. NumPy, SciPy and Numerical Code](#10-numpy-scipy-and-numerical-code)
- [11. Performance and Memory](#11-performance-and-memory)
- [12. Copies, Views and Array Ownership](#12-copies-views-and-array-ownership)
- [13. Public API Design](#13-public-api-design)
- [14. I/O, Plotting and User Interaction](#14-io-plotting-and-user-interaction)
- [15. File and Path Handling](#15-file-and-path-handling)
- [16. Imports and Dependencies](#16-imports-and-dependencies)
- [17. Cython](#17-cython)
- [18. Testing and Correctness](#18-testing-and-correctness)
- [19. Comments and Docstrings](#19-comments-and-docstrings)
- [20. Tooling](#20-tooling)

---

## Quick Rules for New Contributors

If you are new to the project, start here:

1. Make the code correct first.
2. Design numerical code with data layout and performance in mind from
   the beginning.
3. Make the public API simple and unsurprising.
4. Use Ruff and an 88 character line length.
5. Type hint all function arguments and return values.
6. Use descriptive names:
   functions are verbs, classes are nouns.
7. Prefer functions, dataclasses and composition over inheritance.
8. Do not use clever Python features when an explicit alternative exists (e.g.
   @property vs an explicit function).
9. Do not mutate arguments unless the function name makes that behaviour obvious.
10. Prefer NumPy/SciPy operations over Python loops across numerical values.
11. Document NumPy array shapes and axis meanings.
12. Keep computation separate from I/O, plotting and user interaction.
13. Library functions should normally be silent.
14. Fail quickly rather than continuing with questionable data.
15. Every bug fix gets a regression test.
16. Prefer analytic tests and gold end-to-end regression tests.
17. Keep public imports shallow:
    normally no deeper than `package.submodule.thing`.
18. Avoid unnecessary dependencies beyond NumPy, SciPy, Matplotlib and Cython.

---

## 1. Formatting

- Use **Ruff** for formatting and everyday linting.
- Use an **88 character line length**.
- Follow [PEP 8](https://peps.python.org/pep-0008/) unless this guide explicitly says otherwise.
- Use blank lines to separate logical groups of statements.
- Prefer a few clear intermediate statements over one dense expression.
- Keep `if` conditions simple.
  Calculate complex predicates in named intermediate variables before the `if`.
- Keep comprehensions simple:
  normally one line, one `for` loop and at most one function call.
  Use explicit statements or loops for filters, nested loops,
  nested comprehensions or multiple operations.

Example:

```python
is_inside: bool = x >= xmin and x <= xmax
is_valid: bool = value is not None

if is_inside and is_valid:
    ...
```

---

## 2. Naming

Use descriptive names.
Code should normally explain itself without requiring comments.

### Functions

Function names use `snake_case` and should start with a **verb** that
describes what the function does.

```python
calculate_error()
transform_mesh()
load_image()
extract_surface()
```

Avoid vague names such as `process()`, `handle()` or `do_thing()` unless the meaning is obvious from
context.

### Classes

Class names use `PascalCase` and should normally be **nouns**.

```python
Camera
Mesh
CalibrationResult
```

For related type families, put the major concept first:

```python
FieldScalar
FieldVector
FieldTensor
```

rather than:

```python
ScalarField
VectorField
TensorField
```

### Interfaces

Prefix abstract interfaces (ABCs) with a capital `I`:

```python
ISensor
IRenderer
IField
```

### Enumerations

Prefix enumeration classes with a capital `E`:

```python
EGeneratorType
EInterpolationType
```

### Constants

Module constants use `UPPER_SNAKE_CASE`:

```python
DEFAULT_TOLERANCE = 1.0e-8
MAX_ITERATIONS = 100
```

Avoid magic numbers.
Give important numerical values descriptive names and add a comment when
the name alone does not explain the value.

### Acronyms and abbreviations

Common, unambiguous abbreviations are fine.

```python
calc_error
img_shape
coord_index
```

Use lowercase acronyms in functions and variables:

```python
dic_error
rgb_image
calculate_psf()
```

Keep acronyms uppercase in class names:

```python
DICErrorModel
RGBImage
PSFModel
```

Single-letter names are only acceptable for obvious indices or iterators.
NumPy-style iterator names such as `ii`, `jj` and `kk` are fine.

---

## 3. Type Hints

Type hint everything that forms part of an interface and anything where
the type helps the reader understand the code.

- Type hint all function arguments.
- Type hint all function return values.
- Type hint public class attributes.
- Type hint local variables when the concrete type is not obvious or
  is useful context.
- Avoid `Any` unless interacting with genuinely dynamic external code.
- Use modern union syntax such as `Path | None`.
- Prefer a real `Enum` over `Literal` when values represent a meaningful finite set.

Examples:

```python
def render(
    scene: Scene,
    camera: Camera,
    config: RenderConfig,
    ouput_dir: Path | None = None,
) -> RenderResult | None:
    ...
```

This is useful even when the type checker could infer the type because
the annotation tells the reader immediately what `create_image()` returns.

```python
image_path: Path = create_image(...)
```

---

## 4. Functions and Control Flow

- Use **guard clauses** to reduce nesting.
- Fail fast with clear error messages.
- Check everything cheap that can be checked before starting expensive work.
- Do not use exceptions for normal control flow.
- In most scientific applications an unrecoverable error should raise an exception and
  stop the calculation rather than trying to continue in an uncertain state.
- A large function is not automatically bad.
  A function that forces the reader to jump through several layers of
  unnecessary helpers can be worse.
- Functions with a single call site should be avoided and the logic inlined.
- Long argument lists are acceptable when they make important inputs explicit.
- Use keyword only arguments for optional or configuration like parameters.
- Avoid positional booleans.
- Use an `Enum`, `Thing | None`,
  or another meaningful type when it communicates intent better
than a boolean.

An example function definition is shown below:

```python
def render(
    scene: Scene,
    camera: Camera,
    *,
    interp: EInterpType,
    psf: PSF | None = None,
) -> RenderResult:
    ...
```

Prefer this over:

```python
def render(scene, 
           camera, 
           interp_on=True, 
           interp_type=EInterpType.linear, 
           psf_on=False, 
           psf_type=None):
```

Note that the booleans here are duplicating information that can just be expressed in
the `interp_type` and the `psf_type` directly.

---

## 5. Errors, Warnings and Validation

Prefer explicit failure over silent recovery.

- Raise clear exceptions when inputs are invalid, ambiguous or unsafe.
- Do not silently guess when a wrong assumption could change the physical or
  numerical meaning of a result.
- Treat warnings as failures in normal project development and testing.
- Keep `try` blocks as small as possible.
- Catch only exceptions that are expected and can be handled meaningfully.
- Never use bare `except:`.
- Do not catch an exception simply to continue with potentially invalid results.
- Use explicit exceptions for user and input validation.
- Use `assert` only for programmer invariants that should never fail in valid code.

---

## 6. Data Structures

Prefer explicit, typed data structures over flexible containers.

### Dataclasses

Prefer dataclasses over dictionaries for structured data and configuration.

```python
@dataclass(slots=True)
class CameraConfig:
    width: int
    height: int
    bit_depth: int = 8
```

The reason for this is that:

- required fields are explicit;
- defaults are visible;
- arbitrary keys cannot be added accidentally;
- type checkers can reason about the structure;
- `slots=True` prevents dynamic attributes and reduces memory overhead.

Dataclasses should primarily contain **data**.
Validation and simple setup in `__post_init__()` are fine,
but substantial algorithms should normally live in functions or
behavioural classes.

For mutable defaults, use `None` or `field(default_factory=...)` as appropriate.
Never use a mutable object directly as a default value.

### Structured return values

Prefer a named dataclass over a large tuple return.

```python
@dataclass(slots=True)
class CalibrationResult:
    intrinsics: np.ndarray
    extrinsics: np.ndarray
    residuals: np.ndarray
```

This is clearer than returning a tuple whose element meanings must be
remembered.

---

## 7. Classes, Abstraction and Inheritance

Use a mixture of plain functions and classes.
Do not use object oriented programming simply because Python supports it.

### Prefer

- plain functions;
- dataclasses;
- composition;
- dependency injection;
- abstract base classes when an interface is genuinely useful.

### Inheritance

Inheritance is only for a **pure abstract interface** using `ABC`.

- No multiple inheritance.
- No mix-ins.
- Do not inherit from multiple interfaces.
- Keep abstraction to one layer.
- Prefer composition and dependency injection.

Introduce an interface only when it solves a real problem.
A useful rule of thumb is to add one when there are around three implementations or
when explicit conditional dispatch has become genuinely awkward.

### Normal classes

For non-dataclass classes, use `__slots__` where practical:

```python
class Solver:
    __slots__ = ("tolerance", "max_iterations")
```

### Class methods

`@classmethod` is acceptable for clear alternative constructors:

```python
@classmethod
def from_file(cls, path: Path) -> Self:
    ...
```

`@staticmethod` is also acceptable where it genuinely improves organisation.

---

## 8. Restricted and Banned Python Features

Python contains many powerful features that are unnecessary for
most engineering software.
Avoiding them makes the code easier to read, debug and maintain.

### Use normally when appropriate

- functions;
- dataclasses;
- enums;
- type hints;
- context managers;
- abstract base classes;
- generators;
- `@classmethod` for alternative constructors;
- `@staticmethod` where appropriate.

### Use sparingly

- decorators;
- lambdas;
- generators where a simpler structure is clearer.

Use lambdas only for trivial single expression callbacks or sort keys.
Give meaningful behaviour a named function.

### Banned

Do not use:

- operator overloading;
- multiple inheritance;
- mix-ins;
- `@property`;
- metaclasses;
- monkey patching;
- custom `__getattr__`;
- custom `__getattribute__`;
- custom `__setattr__`;
- double-underscore name mangling;
- dynamic attribute creation;
- wildcard imports;
- `eval()`;
- `exec()`.

Do not invent clever object semantics through custom dunder methods when
a standard data structure or explicit method would be clearer.

---

## 9. Argument Mutation and Side Effects

Functions must not silently mutate arguments supplied by the caller.
If a function mutates an input, the mutation must be explicit in the API by using one of
the following conventions:

By explicitly returning the mutated input:

```python
def transform_array(array: np.ndarray) -> np.ndarray:
    array *= 2.0
    return array

array = transform_array(array)
```

or by using an `out=` keyword argument:

```python
def transform_array(
    *,
    out: np.ndarray,
) -> None:
    out *= 2.0

transform_array(out=array)
```


Returning a mutated object does not create a copy.
The returned value is another reference to the same object.
For performance sensitive numerical code, prefer an `out=` argument when
the caller may benefit from controlling allocation or reusing existing storage.

Mutation of `self` by instance methods is exempt from this rule because
modifying object state is an expected part of method semantics:

```python
class Camera:
    def set_position(self, position: np.ndarray) -> None:
        self.position = position
```

Do not use mutable global state.
Module level constants are fine;
mutable module level configuration and caches should be avoided unless there is
a strong justification.

---

## 10. NumPy, SciPy and Numerical Code

NumPy and SciPy are the default tools for numerical work.

- Prefer NumPy and SciPy operations over Python loops across individual numerical
  values.
- Push numerical work into compiled operations where this makes the code clear and
  fast.
- Ordinary Python loops over high-level objects are fine.
- Do not contort simple control flow merely to remove a loop.

Good:

```python
scaled = values * scale
```

Avoid:

```python
for ii in range(values.shape[0]):
    scaled[ii] = values[ii] * scale
```

Also good:

```python
for camera in cameras:
    render_camera(camera)
```

### Array meaning must be explicit

NumPy arrays are opaque.
When manipulating important arrays, document the following in comments and
docstrings:

- shape;
- axis meaning;
- dtype;
- coordinate convention;
- indexing convention where relevant.

Example:

```python
# coords: (num_nodes, 3)
# axis 0 -> node
# axis 1 -> x, y, z coordinates
```

or:

```python
# images: shape=(num_frames, height, width, num_channels)
```

Document these details thoroughly in public docstrings and use short local comments where
array shapes or axis meanings would otherwise be unclear during manipulation.

### Units

Numerical code is normally **unitless**, as in many finite element codes.
The caller is responsible for supplying a consistent system of units.
Do not silently convert or assume mixed units unless a particular API explicitly exists for
that purpose.

---

## 11. Performance and Memory

Design with performance intent from the beginning,
then measure and improve.

Do not deliberately write an inefficient implementation on the assumption that
profiling can fix it later.
Poor data layout can make later optimisation difficult or
require a complete rewrite.

When designing numerical code, consider:

- data layout;
- array shape and axis order;
- access patterns;
- cache behaviour;
- contiguous memory access;
- allocation and copying;
- opportunities for vectorisation;
- the cost of reshaping, transposing and temporary arrays.

A fast implementation starts with a sensible data representation.

Once the basic structure is sound:

- profile the implementation;
- measure real bottlenecks;
- improve the expensive parts;
- do not destroy readability for insignificant gains.

Be aware that apparently simple NumPy operations may allocate temporary arrays or
copies.
Avoid unnecessary allocations in performance sensitive code.

---

## 12. Copies, Views and Array Ownership

Be explicit about whether an operation returns a copy or a view.

- Avoid unnecessary copies in performance sensitive code.
- Prefer a copy when shared memory would make behaviour surprising or unsafe.
- Do not rely on subtle NumPy view behaviour being obvious to the caller.
- Document unusual view or ownership behaviour.
- When a function intentionally modifies a shared array,
  make that behaviour obvious from the API and documentation.

Correctness and predictability are more important than avoiding every copy.

---

## 13. Public API Design

The public API should be easy to remember and difficult to misuse.

### Keep APIs shallow

The deepest public API should normally be:

```text
package.submodule.thing
```

Typical usage should look like:

```python
from pyvale import render

render.mesh_transform(...)
render.cam_look_at(...)
```

Avoid forcing users to remember deeply nested paths such as:

```python
pyvale.render.geometry.transforms.mesh_transform(...)
```

Internal modules can be deeper.
Use `__init__.py` re-exports to lift public functionality to the appropriate level.

### Make important data obvious

The major data that determines what a function does should be visible in
the function signature.

Do not hide major inputs behind unnecessary layers of configuration simply to
shorten the argument list.

### Defaults

Provide defaults when there is a safe and unsurprising choice.
Require the caller to provide a value when choosing a default could silently
change the physical or numerical meaning of the calculation.

Default tolerances on floating point calculations should be named `CONSTANTS` and the reason for
the selected value should be documented.

### Predictability

A function should do what its name suggests and nothing surprising.

Prefer:

```python
results = calculate_error(...)
plot_error(results)
```

rather than having `calculate_error()` unexpectedly open a plot window.

---

## 14. I/O, Plotting and User Interaction

Separate:

- computation;
- file I/O;
- plotting;
- user interaction.

Prefer:

```python
results = calculate_stress(...)
save_stress(results, path)
plot_stress(results)
```

rather than one function that calculates, saves, prints and plots.

Most library functions should be silent and should not print to the console.

For long-running simulations or calculations,
progress and status output can be useful,
but there must always be an explicit way to disable it.

For example:

```python
run_simulation(..., show_progress=False)
```

---

## 15. File and Path Handling

Use `pathlib.Path` for filesystem paths and file I/O.

Prefer:

```python
path = Path("results") / "image.png"
```

Do not manually build paths with string concatenation and avoid `os.path` for
normal path handling.

Keep file I/O outside numerical kernels wherever practical.

---

## 16. Imports and Dependencies

### Imports

Module qualified imports are encouraged because they preserve context:

```python
import numpy as np
import scipy as sp
import matplotlib.pyplot as plt
from pyvale import render
```

Avoid wildcard imports:

```python
from module import *
```

### Core scientific dependencies

Treat the following as our normal scientific Python platform:

- NumPy;
- SciPy;
- Matplotlib;
- Cython.

Prefer these tools for scientific and numerical work.
After this core set, be conservative about adding third-party dependencies.
Add another dependency only when it provides substantial functionality that
would be costly, risky or distracting to implement ourselves.
Do not add dependencies for trivial convenience functions.

---

## 17. Cython

Use Cython when Python and NumPy are no longer sufficient for
performance sensitive code.

Prefer modern **pure Python mode** Cython syntax where practical.

In Cython:

- explicit loops are fine and often desirable;
- type numerical values and arrays clearly;
- think about memory layout and access patterns;
- minimise Python interaction inside hot loops;
- avoid unnecessary allocation;
- be conscious of bounds checking and other runtime overhead;
- keep the Python API simple even when the implementation is specialised.

Do not mechanically vectorise Cython code simply because loops are discouraged in
normal Python.
The point is to move expensive iteration into compiled code.

---

## 18. Testing and Correctness

Testing is part of the implementation and enforces **Make it correct**.
Tests should provide confidence in behaviour and numerical correctness without
constraining the implementation.

### Every bug gets a regression test

Every reproducible bug fix must include a test that fails before the fix and
passes after it.

### Prefer independent correctness tests

Where possible, test against:

- analytic solutions;
- invariants;
- conservation laws;
- independently calculated values;
- manufactured solutions;
- known limiting behaviour.

Prefer tests that provide an independent correctness oracle over tests that
reproduce the implementation logic inside the test.

### Use gold regression tests

Gold regression tests are strongly encouraged for
end-to-end numerical workflows.
A gold file should represent a known correct result that does not change merely because
the API or implementation was refactored.
Use a strong balance of:

- analytic or independently derived tests;
  and
- end-to-end gold regression tests.

### Test behaviour, not implementation

Tests should normally verify externally observable behaviour rather than
implementation details.
Internal refactoring should not break tests when
the public behaviour remains correct.
Avoid tests that reach unnecessarily into private functions, internal state,
intermediate representations,
or implementation specific call sequences.
Testing internal components directly is appropriate where they implement substantial or
independently meaningful behaviour.

### Avoid low-value and redundant tests

Do not add tests merely to exercise code that has no meaningful behaviour to
verify.
For example, a trivial constructor normally does not require a dedicated test if
it simply stores its arguments and cannot fail:

```python
class Camera:
    def __init__(self, width: int, height: int) -> None:
        self.width = width
        self.height = height
```

Test construction when it performs validation, transformation,
resource allocation,
or other behaviour that can meaningfully succeed or fail.

Avoid:

- repetitive tests that exercise the same behaviour through slightly different inputs without
  adding useful coverage;
- multiple tests of the same behaviour at different layers unless each provides a
  distinct correctness guarantee;
- tests that merely confirm removed functionality, classes, functions,
  or attributes are absent;
- tests that reproduce implementation logic rather than independently checking its
  result;
- tests coupled to private implementation details that
  may legitimately change during refactoring;
- tests whose only purpose is to increase line or branch coverage.

Parameterisation should be used when several cases exercise the same behaviour:

```python
@pytest.mark.parametrize(
    ("value", "expected"),
    [
        (0.0, 0.0),
        (1.0, 2.0),
        (-1.0, -2.0),
    ],
)
def test_scale_value(value: float, expected: float) -> None:
    assert scale_value(value) == expected
```

Separate tests are preferred when different cases represent meaningfully
different behaviours or failure modes.

### Use fixtures for setup and teardown

Use `pytest.fixture` for test setup and teardown when a test creates temporary files, directories,
resources,
or other state that must be cleaned up.
Prefer a fixture using `yield` so teardown runs even if the test raises an exception or
an assertion fails:

```python
from collections.abc import Iterator
from pathlib import Path

import pytest


@pytest.fixture
def output_file(tmp_path: Path) -> Iterator[Path]:
    path = tmp_path / "output.dat"

    yield path

    path.unlink(missing_ok=True)
```

Use the fixture explicitly in tests:

```python
def test_write_output(output_file: Path) -> None:
    write_output(output_file)

    assert output_file.exists()
```

Do not place cleanup only at the end of the test body:

```python
def test_write_output(tmp_path: Path) -> None:
    path = tmp_path / "output.dat"

    write_output(path)
    assert path.exists()

    # Do not rely on cleanup here.
    path.unlink()
```

If the test fails before reaching the cleanup code, the teardown will not run.
Prefer built-in pytest fixtures such as `tmp_path` where
they already provide the required lifecycle management.

### Keep tests clear and focused

Use descriptive test names and keep each test focused on a clear behaviour.
A test should make it obvious:

- what behaviour is being exercised;
- what result is expected;
  and
- why failure indicates a problem.

---

## 19. Comments and Docstrings

### Comments

Use comments sparingly.
Prefer code that explains itself.

Comments are useful for:

- explaining why a non-obvious decision was made;
- numerical assumptions;
- array shapes and axis meanings;
- algorithmic subtleties;
- references to papers or standards;
- temporary workarounds that need explanation;
- clearly marking sections of long or complex code with header style blocks.

Do not narrate obvious code:

```python
# Increment ii
ii += 1
```

### Docstrings

Public docstrings should be thorough.

Use **NumPy-style docstrings** for project code.

Document:

- purpose;
- parameters;
- return values;
- array shapes;
- axis meanings;
- dtypes where important;
- assumptions;
- expected unit consistency;
- raised exceptions;
- important numerical behaviour;
- references to papers, standards or algorithms where relevant.

Use autodocstring tooling where helpful.

---

## 20. Tooling

Use the following standard development tools:

- **Ruff** —
  formatting and everyday linting;
- **Pyright** —
  static type checking;
- **Pylint** —
  optional deeper linting and code-quality review;
- **pytest** —
  testing.

A useful mental model is:

- Ruff keeps the code tidy.
- Pyright checks the types.
- pytest checks the behaviour.
- Pylint can provide a deeper review when needed.

