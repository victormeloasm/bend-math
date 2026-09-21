# bend-math

A comprehensive `F32` mathematical library for [Bend 2](https://github.com/bendlang/bend), modeled after the C `math.h` API.

`bend-math` provides the usual trigonometric, hyperbolic, exponential, logarithmic, power, rounding, remainder, gamma, error-function and IEEE-754 helper operations, together with traditional mathematical constants such as `M_PI`, `M_E` and `M_SQRT2`.

The implementation is designed around Bend's current numeric model rather than pretending Bend has C types that do not exist.

> Upstream integration is currently proposed in [bendlang/bend#929](https://github.com/bendlang/bend/pull/929).

## Features

* C99/C11-style `math.h` functionality for Bend's native `F32`
* additional C23-style functions
* IEEE-754 classification and bit-level helpers
* explicit handling of NaN, infinities, signed zero and subnormals
* fused F32 FMA support
* stable implementations of numerically sensitive operations
* C and JavaScript backend support
* regression and edge-case tests
* traditional `math.h` constants
* no dependency on an F64 type inside Bend code

## Constants

Bend does not have C-style value macros, so the usual `M_*` constants are exposed as nullary definitions:

```bend
M.M_E()
M.M_LOG2E()
M.M_LOG10E()
M.M_LN2()
M.M_LN10()

M.M_PI()
M.M_PI_2()
M.M_PI_4()
M.M_1_PI()
M.M_2_PI()
M.M_2_SQRTPI()

M.M_SQRT2()
M.M_SQRT1_2()
```

Example:

```bend
import ./src/math.bend as M

def main():
  IO.print(F32.show(M.M_PI()))

#|3.1415927
```

## API

### Classification and IEEE-754 helpers

```text
bits
signbit
isnan
isinf
isfinite
isnormal
fpclassify

isgreater
isgreaterequal
isless
islessequal
islessgreater
isunordered

fabs
copysign
fmin
fmax
fdim
nextafter
nexttoward
```

### Roots, powers and scaling

```text
sqrt
cbrt
rsqrt
hypot

pow
pown
powr
rootn

scalbn
scalbln
ldexp
frexp
```

`powr` follows the C23 argument order:

```bend
# powr(y, x) = x^y

M.powr(3.0, 2.0)

# result: 8.0
```

### Exponential and logarithmic functions

```text
exp
exp2
exp10
expm1

exp2m1
exp10m1

log
log2
log10
log1p

logp1
log2p1
log10p1

logb
ilogb
```

### Trigonometric functions

```text
sin
cos
tan

asin
acos
atan
atan2

sincos
```

### Pi-scaled trigonometric functions

```text
sinpi
cospi
tanpi

asinpi
acospi
atanpi
atan2pi
```

These functions avoid the unnecessary argument reduction error that would result from simply evaluating expressions such as:

```text
sin(x * PI)
```

For example, integer inputs to `sinpi` preserve the expected exact-zero behavior.

### Hyperbolic functions

```text
sinh
cosh
tanh

asinh
acosh
atanh
```

### Rounding

```text
ceil
floor
trunc

round
roundeven
nearbyint
rint

lround
llround
lrint
llrint
```

### Remainder and decomposition

```text
fmod
remainder
remquo
modf
frexp
```

### Error and gamma functions

```text
erf
erfc
tgamma
lgamma
```

### Fused multiply-add

```bend
M.fma(a, b, c)
```

This implements fused multiply-add semantics rather than simply evaluating:

```text
a * b + c
```

which may produce a different F32 result because of intermediate rounding.

## Example

```bend
import Base
import ./src/math.bend as M

def main():
  pi   = M.M_PI()
  root = M.sqrt(2.0)
  s    = M.sinpi(0.5)
  g    = M.tgamma(5.0)

  IO.print(
    F32.show(pi)
    ++ " "
    ++ F32.show(root)
    ++ " "
    ++ F32.show(s)
    ++ " "
    ++ F32.show(g)
  )
```

Expected values are approximately:

```text
3.1415927 1.4142135 1 24
```

## Bend-specific integer type

Bend currently has no native signed integer type equivalent to C's `int`, `long` or `long long`.

For APIs that require a signed integer, `bend-math` provides:

```bend
type S32 is Data:
  SNeg{mag: U32}
  SPos{mag: U32}
```

This is deliberately explicit.

The library does not pretend Bend has native signed 64-bit storage when it does not.

For example:

```bend
M.pown(2.0, M.S32.pos(10))
```

computes:

```text
1024.0
```

`llround`, `llrint`, `scalbln`, `pown`, `rootn` and related interfaces therefore use Bend-compatible integer representations rather than being binary-compatible copies of the corresponding C signatures.

## Floating-point environment

Bend currently has no C-style `fenv`, floating-point exception flags or `errno`-based math error environment.

Because of this:

```bend
M.math_errhandling()
```

returns `0`.

`rint` and `nearbyint` use deterministic round-to-nearest-even behavior instead of depending on a process-wide floating-point rounding mode.

## Installation

### With upstream support

Once [bendlang/bend#929](https://github.com/bendlang/bend/pull/929) is merged, the required backend support should be available directly in Bend.

The standalone module can then be copied into your project and imported normally.

For example:

```bash
cp src/math.bend /path/to/my-project/math.bend
```

Then:

```bend
import ./math.bend as M
```

### Current Bend versions

Until the upstream integration is available, this repository contains the compiler/runtime integration patch required by the library:

```text
patches/bend-integration.patch
```

From a clean Bend checkout:

```bash
git apply /path/to/bend-math/patches/bend-integration.patch
```

The patch contains the backend primitives and Base support required by `math.bend`.

## Repository layout

```text
bend-math/
├── src/
│   └── math.bend
│
├── backend/
│   └── math.ts
│
├── tests/
│   ├── math_edges.bend
│   ├── math_fma.bend
│   ├── math_h.bend
│   ├── math_regressions.bend
│   └── math_release.bend
│
└── patches/
    └── bend-integration.patch
```

`src/math.bend` contains the public Bend library.

`backend/math.ts` contains backend implementations for operations that cannot be reproduced correctly or efficiently using only existing F32 primitives.

The integration patch wires those operations into Bend.

## Validation

The implementation has been tested using both randomized differential tests and explicit IEEE-754 edge cases.

The release audit included:

```text
3,300,000 randomized C <-> JavaScript comparisons
5,355 explicit IEEE-754 edge cases
UBSan
float-cast-overflow sanitizer
ASan
LeakSanitizer
Bend checker regression testing
C backend execution
JavaScript backend execution
```

The test corpus covers cases including:

```text
+0 / -0
subnormals
minimum normal values
maximum finite F32
+infinity / -infinity
NaN
integer and half-integer boundaries
extreme exponents
FMA rounding
scaling
gamma functions
error functions
pi-scaled trigonometry
integer powers
remainder/remquo
```

The repository gate used by Bend also passes after the backend code was separated from `comp.ts` to remain below the project's source-size gate.

## Numerical design

Some functions can be implemented directly in Bend without sacrificing correctness.

Others cannot.

Operations such as `expm1`, `log1p`, FMA, gamma functions and certain C23 operations require care because naive composition of existing F32 operations introduces intermediate rounding, cancellation or premature underflow/overflow.

Where necessary, `bend-math` uses backend implementations so the operation is evaluated with a wider intermediate representation and rounded to F32 only at the appropriate point.

The goal is not merely to provide functions with familiar names.

The goal is for those functions to behave like mathematical primitives.

## Backend status

C and JavaScript backends have been directly compiled, executed and differentially tested.

Metal support follows the corresponding backend paths but has not been runtime-tested on Apple hardware by this project.

## Upstream

The Bend integration is being proposed upstream here:

[bendlang/bend#929](https://github.com/bendlang/bend/pull/929)

Main Bend repository:

[bendlang/bend](https://github.com/bendlang/bend)

## Author

**Victor Duarte Melo (Fr0ggy)**

Security researcher, programmer and occasional frog-powered numerical library maintainer. 🐸
