---
title: "One in 2^62: When `(int)(x + 0.5)` Rounds the Wrong Way"
author: ntysdd
pubDatetime: 2026-10-09T09:12:41Z
modDatetime: 2026-10-21T14:44:35Z
description: |
 `(int)(x + 0.5)` returns the wrong integer for exactly one non-negative double whose cast to
 `int` is defined, and for four million non-negative floats. The `long long` version is wrong for
 half of the nanosecond timestamps in one `2^52`-wide window.
tags:
  - Floating-Point
  - C

---

> **The short version.** `(int)(x + 0.5)` is wrong for `0.49999999999999994` (it returns `1`)
> and for four million floats. It is also wrong for every negative `x` below `-0.5` that is not a
> half-integer, because `(int)` truncates where it should floor. `(long long)(x + 0.5)` is wrong
> for `2^51` doubles in `[2^52, 2^53)`, which a nanosecond counter enters after about 52 days and
> leaves 52 days later. `lround` never adds at all; some Windows toolchains link a copy that is
> not exact, and it gets this input wrong too.

`(int)(x + 0.5)` is how you round without calling a library. C89 had no `round`, and FORTRAN 66
had no nearest-integer intrinsic until FORTRAN 77 added `NINT`, which probably explains why the
idiom turns up in old Fortran too. `NINT` itself was specified as `INT(A + 0.5)` for positive `A`,
and it took J3 interpretation 000012 to rule that such wording describes the result rather than
requiring the computation. Nothing below is C-specific: the languages differ, but each of them
hands you a truncating conversion, and the failures depend on whether the value is a `float` or a
`double` and on the precision the addition is evaluated in.

The canonical counterexample is `x = 0.49999999999999994`:

```c
double x = 0x1.fffffffffffffp-2;   /* 0.49999999999999994 */
printf("%d\n", (int)(x + 0.5));    /* 1 */
```

It prints `1`; the nearest integer is `0`. Nothing overflowed, no library call was involved, and
`0.5` is exactly representable. The addition rounded.

All counts below are for non-negative inputs where the cast to `int` is defined,
`0 <= x < 2^31 - 0.5`:

| `x` | addition evaluated in | wrong inputs |
|---|---|---|
| `float` | `float` | 4,194,305 of 1,325,400,064 (0.32%) |
| `float` | `double` | **0** |
| `double` | `double` | **1** |
| `double` | 80-bit `long double` | 0 |

The last row is derived rather than measured: at 64 bits the failure window would be
`S ∈ [1 - 2^-65, 1)`, which needs `x ∈ [0.5 - 2^-65, 0.5)`, and no double lives there. Anywhere
else a rounded sum still cannot cross an integer. The other three rows are exhaustive over that
range.

The rest of the post walks through where those numbers come from. The counterexamples are old
and I cite the prior write-ups below; the part I have not seen stated is that the two mechanisms
behind the table are the only ones, for each of the four pairings of input format and addition
precision. The proof and the measurement log are in the appendices, and if you only want a
drop-in replacement, skip to [What to write instead](#what-to-write-instead).

## The one bad double

`0x1.fffffffffffffp-2` is the largest double below `0.5`. It is `0.5 - 2^-54`, and that `2^-54`
is exactly half the spacing of doubles just below `1.0`. Add `0.5` and the exact result is
`1 - 2^-54`, the midpoint between `0x1.fffffffffffffp-1` (the largest double below one) and
`1.0`. IEEE's default rule for a tie is round-half-to-even, and `1.0` has an even significand,
so the sum lands on `1.0`. Then `(int)` truncates it to `1`.

The cast is not the problem; the addition is already wrong by the time the cast runs.
`floor(x + 0.5)` returns `1.0` as well:

```c
double x = 0x1.fffffffffffffp-2;
floor(x + 0.5);            /* 1.0  -- the sum is already 1.0 */
floor(x) + (x - floor(x) >= 0.5);   /* 0    -- no rounding at all */
```

The exact sum needed one bit (`2^-54`) that a double cannot store, and that is the only bit the
truncation cares about.

## Why there is only one

"Exactly one" means nothing without a range, so: over every double in `[0, 2^31 - 0.5)`, under
the default round-to-nearest, `(int)(x + 0.5)` disagrees with `floor(x + 0.5)` at exactly one
value.

The reason takes three steps, and the only quantity that matters is `ulp(x)`, the spacing
between adjacent doubles at `x`. Inside `int` range that spacing is below `1` everywhere
(`ulp(x) >= 1` needs `x >= 2^52`), so `x + 0.5` is exact whenever the sum stays on the grid
that `x` sits on.

1. If `x` and `x + 0.5` sit in the same *binade* (the stretch between two consecutive powers of
   two, where the spacing is constant), then `0.5` is an exact multiple of `ulp(x)` and the sum
   is exact. So the sum can only round when it crosses a power of two.
2. Crossing means `x` is just below some `2^j`, and the sum lands in `[2^j, 2^j + 0.5)`. For
   `j >= 0` the next integer is `2^j + 1`, a full `0.5` away, and rounding can move the sum by
   at most half a spacing (`2^-23` or less inside `int` range). For `j <= -2` the sum is below
   `0.75` and the next integer is out of reach.
3. That leaves `j = -1`: the sum lands in `[0.5, 1)`. To be rounded up to `1` it has to reach
   `1 - 2^-54`, the midpoint of that binade, and no double can hold that midpoint; that is the
   whole problem. Exactly one `x` produces it: `0.5 - 2^-54`, the last double before `0.5`. The
   tie goes to even, so the sum becomes `1.0`.

So `x = 0.5 - 2^-54` is the only input that reaches step 3. The same three steps work for any
significand width, which is why the `float` count is different but comes from the same place.

The neighbours of the bad input, for orientation:

| `x` | `x + 0.5` | `(int)(x+0.5)` | correct |
|---|---|---|---|
| `0x1.ffffffffffffep-2` | `0x1.fffffffffffffp-1` | 0 | 0 |
| **`0x1.fffffffffffffp-2`** | **`0x1p+0`** | **1** | **0** |
| `0x1p-1` | `0x1p+0` | 1 | 1 |

This is why the idiom survived four decades of production code. Sampling doubles uniformly over
the bit patterns hits it with probability on the order of `2^-62`: there are about 4.7 × 10^18
non-negative doubles below `2^31`, exactly one of them returns a wrong answer, and the few above
`2^31 - 0.5` return nothing at all. A test suite would have to target it explicitly to find it.

None of this is new. The `float` half was written up in 2013 by Pascal Cuoq on the Frama-C blog,
in [*Harder than it looks: rounding float to nearest integer*](https://www.frama-c.com/2013/05/02/Harder-than-it-looks-rounding-float-to-nearest-integer-part-1.html):
the `ulp == 1` range, the count ("half these 2^23 values, that is nearly 4 millions of them")
and the `0.49999997f` boundary input, with a working fix at the end. The `double` input is
equally well known: [JDK-6430675](https://bugs.openjdk.org/browse/JDK-6430675), Eric Lippert's
[post](https://ericlippert.com/2013/05/16/spot-the-defect-rounding-part-two/), ridiculousfish on
[JavaScript's `Math.round`](https://ridiculousfish.com/blog/posts/JavaScript-Rounding.html), and
the [Stack Overflow thread](https://stackoverflow.com/questions/9902968/) that connects them.

The mechanisms above are also all of them, which is the claim I have not found in print, and why
the argument is spelled out instead of cited. The case analysis also runs as code, in exact
rational arithmetic over every binade up to `2^31`, and it finds one non-empty window; both
versions are in [Appendix A](#appendix-a-the-case-split-in-full). If someone has written the
enumeration up, I would like the reference.

## Changing `x` to `float`

Two things change. Both are discussed by Cuoq; the counts here are exact rather than estimated.

First, the counterexample is a different number. `0.49999999999999994` is not a float; it rounds
to `0.5f` on the way in, and rounding `0.5f` gives `1` correctly. The float version of that
boundary is `0x3effffff`, the largest float below `0.5`, which is `0.5 - 2^-25`:

```c
float xf = 0x1.fffffep-2f;      /* 0.4999999701976776 */
(int)(xf + 0.5f);               /* 1  -- wrong */
(int)(xf + 0.5);                /* 0  -- right */
```

In C, `xf + 0.5` promotes `xf` to `double` and adds in double precision, where
`0.4999999701976776 + 0.5` is exact: `1 - 2^-25` needs 25 significant bits and double has 53.
The sum is exact there, so nothing goes wrong. The bug needs `0.5f` to keep the addition in
`float`.

That is the one case in this table where the widening happens on its own. A `float` has 24
significand bits, so in double `x + 0.5` is exact wherever the answer depends on it: near `0.5`
the sum needs 25 bits, and in the `ulp == 1` binade it needs 25 as well, well inside 53. For very
small `x` the double sum can still round (`x = 2^-31 + 2^-54` does) but the error is at most
`2^-54` while the nearest integer boundary is `0.5` away, so the cast result is unaffected. C's
promotion rule makes that the default spelling, which is why so much `float` code using this
idiom works.

Second, `float` has an extra failure mode that a `double` input in `int` range cannot reach. In
`[2^23, 2^24)` the spacing of floats is exactly `1`, so every float in that range is an integer.
Add `0.5f` and you get a tie between `x` and `x + 1`, and round-half-to-even picks the even one.
If `x` is odd you get `x + 1`:

```c
float fi = 8388609.0f;          /* 2^23 + 1, exactly representable */
(int)(fi + 0.5f);               /* 8388610  -- one too many */
(int)(fi + 0.5);                /* 8388609  -- right, and exact in double */
```

This one has nothing to do with binade edges. It happens wherever the format's integer precision
(`2^24` for float) is smaller than the integer range of the destination type (`2^31` for `int`).
It is Cuoq's `8388609.0f` example. His estimate, "half these `2^23` values, that is nearly 4
millions of them", is exact up to the boundary input: `2^22` odd integers in `[2^23, 2^24)`, plus
the one from the previous section. Checking every float in `[0, 2^31)` exhaustively splits the
failures cleanly:

```
float32 in [0, 2^31): 1,325,400,064 values
  (int)(x + 0.5f)  wrong: 4,194,305  (0.3165%)
     0x3effffff  0.4999999701976776     <- the binade boundary, one input
     0x4b000001  8388609.0             <- 2^22 odd integers in [2^23, 2^24)
  (int)(x + 0.5)   wrong: 0
```

That is `2^22 + 1`: one input at the binade edge, `2^22` in the integer range.

## When `x` is large

The interesting range is the one binade where `ulp(x)` is exactly `1`.

**Overflow is a non-issue.** Under round-to-nearest, `DBL_MAX + 0.5` rounds straight back to
`DBL_MAX`: the exact sum would have to reach `DBL_MAX + 2^970` to overflow, and `0.5` is smaller
than that gap by almost three hundred decimal orders of magnitude. Directed rounding is the only
way to get an infinity out of this idiom, and it is a real concern if you implement interval
arithmetic, but `x` has to be within an ulp of `DBL_MAX` for it to happen. (`inf` and `NaN`
inputs propagate, and then the cast is undefined anyway.) The measurement is in
[Appendix B](#appendix-b-how-this-was-checked).

**A vanished addition is not a failure.** If `ulp(x) >= 2` then `x` is already an integer and
`round(x) == x`, so adding nothing and truncating to `x` is correct. What breaks is the single
binade where the addition does exactly *half* of something.

For `x >= 0`:

| `ulp(x)` | `x + 0.5` | `(long long)(x + 0.5)` |
|---|---|---|
| `< 1` | rounds normally, or is exact | isolated boundary inputs only |
| `== 1` | exact tie, half-to-even | `x` for even `x`, **`x + 1` for odd `x`** |
| `>= 2` | identical to `x` | `x`, which is right because `x` is an integer |

`ulp(x) == 1` is exactly the binade `[2^(p-1), 2^p)` for a `p`-bit significand: `[2^23, 2^24)`
for `float`, `[2^52, 2^53)` for `double`. One binade wide, and half of it is odd integers:

```
doubles in [2^52, 2^53): 2^52
of which odd:            2^51
all in int64_t range:    yes
```

So `(long long)(x + 0.5)` is one too large for `2^51` values, and every one of them is inside
`int64_t` range, which is why nothing catches it. If those numbers are timestamps, IDs or
nanosecond counters that live in that window, half of them drift up by one. The window is `2^52`
nanoseconds wide, so a monotonic clock starting from zero enters it after 52 days and leaves it
52 days later. With an `int` target the same window sits at `[2^23, 2^24)`, well inside `int`
range: that is the four million from the previous section.

This also explains why the earlier section could claim exactly one bad double. Inside `int`
range, `ulp(x) < 1` everywhere, so the `ulp == 1` window is unreachable and only the boundary tie
below `0.5` is left.

Putting the ranges together, for the two pairings in which the `ulp == 1` window is reachable:

| `x` | `(int)(x + 0.5f)`, `float` x | `(long long)(x + 0.5)`, `double` x |
|---|---|---|
| `[0, 2^23)` / `[0, 2^52)` | 1 bad input (`0x3effffff`) | 1 bad input (`0x1.fffffffffffffp-2`) |
| `[2^23, 2^24)` / `[2^52, 2^53)` | **half of them, 2^22 odd integers** | **half of them, 2^51 odd integers** |
| `[2^24, 2^31)` / `[2^53, 2^63)` | ok, the addition vanishes | ok, the addition vanishes |
| `>= 2^31` / `>= 2^63` | UB | UB |

Every "ok" in the large rows is ok for the same reason: the addition vanishes and the value was
already an integer. The only bad rows are the binades where `ulp == 1`, plus the edges at
`2^31 - 0.5` and `2^63 - 0.5` where the cast itself is undefined. In code:

```c
double x = 4503599627370497.0;  /* 2^52 + 1, odd */
double s = x + 0.5;             /* 4503599627370498.0 -- tie, rounded to even */
(long long)s;                   /* 4503599627370498   -- one too many */

x = 9007199254740991.0;         /* 2^53 - 1, ulp still 1, odd */
s = x + 0.5;                    /* 9007199254740992.0 */
(long long)s;                   /* 9007199254740992   -- one too many */

x = 9007199254740992.0;         /* 2^53, ulp 2, so the 0.5 is simply lost */
s = x + 0.5;                    /* == x */
(long long)s;                   /* correct */
```

Both mechanisms are ties. In both, the exact sum lands exactly halfway between two representable
values and round-half-to-even settles it: below `0.5` the tie is between `1 - 2^-53` and `1`, and
in the `ulp == 1` binade it is between `x` and `x + 1`. `0.5` can produce both because it is an
exact multiple of the spacing just below it, `2^-54`, and exactly half the spacing in the
`ulp == 1` binade.

## Wider intermediates

With 64 bits of significand, `1 - 2^-54` is representable and the cast sees it:

```c
(int)((long double)x + 0.5L);   /* 0 */
```

That is why x87 looks like a rescue. On 32-bit x86 with x87, `FLT_EVAL_METHOD` is 2 and `double`
expressions are evaluated in 80-bit registers, so the addition really does happen at 80 bits.
It does not help, because the cast forces a store back to `double` first:

```
fadd    QWORD PTR .LC0[rip]     ; + 0.5, at 80 bits: exact
fstp    QWORD PTR 8[rsp]        ; store back as a double: 2^-54 is gone
cvttsd2si eax, QWORD PTR 8[rsp]
```

Measured under `-mfpmath=387` with GCC 13: `FLT_EVAL_METHOD=2`, `(int)(x+0.5) = 1`. Extra
precision only helps if it survives to the point where the information is consumed, and a
conversion to `int` is exactly where the compiler is entitled to narrow first.

This is also why the `q >= 2p + 2` rule from my `powf(x, 3)` post does not apply: it protects a
value from being rounded a second time, and here a single rounding already destroys the bit the
next step needs.

## Two more ways it fails

**Truncation is not floor.** `(int)` rounds toward zero. When `x + 0.5 < 0` that is a ceiling of
the sum instead of a floor, so for `x < -0.5` the idiom is off by one except where the sum is
exactly an integer. The half-integers come out right:

```c
(int)(-0.5 + 0.5);   /*  0  -- sum is 0, exact */
(int)(-1.5 + 0.5);   /* -1  -- sum is -1, exact */
(int)(-2.5 + 0.5);   /* -2  -- sum is -2, exact */
(int)(-2.1 + 0.5);   /* -1  -- sum is -1.6; nearest integer is -2 */
```

`(int)(x + 0.5)` equals `floor(x + 0.5)` for `x >= -0.5`, and also at the half-integers below it.
And even `floor(x + 0.5)` is round-half-up (ties toward `+∞`), not the round-half-away-from-zero
that "四舍五入" means in Chinese and that C's `round()` and Fortran's `NINT` implement: half-up
sends `-2.5` to `-2`, while `round()` sends it to `-3`.

**The cast is undefined out of range.** `x + 0.5 >= 2^31` is undefined behavior, not a
saturating conversion; that is the UB rows in the tables above. On x86-64 what you actually get
is `INT_MIN`, and it is silent:

```
(int)(2147483647.5 + 0.5)  = -2147483648
(int)(-2147483649.5 + 0.5) = -2147483648
```

Note that `2147483647.0 + 0.5` is fine: the sum is `2147483647.5`, which truncates to
`INT_MAX`. The first input that is not fine is `2147483647.5` itself, whose sum is
`2147483648.0`. Same for `NaN`, `infinity`, and for `long long` at `2^63`. `x` has to be
checked to be in `[-0.5, 2147483647.5)` before the cast, and the check has to happen in a type
wide enough to express those bounds.

## What to write instead

If you want round-half-away-from-zero, the library already has it, correctly rounded and without
an addition:

```c
#include <math.h>

long  r = lround(x);     /* half away from zero, returns long */
long long q = llround(x);/* same, for 64-bit targets */
```

`lrint`/`llrint`/`rint`/`nearbyint` are faster on most hardware (a single `roundsd`) but round
half to even, so `rint(0.5) == 0` and `rint(2.5) == 2`. That is a different function, so it is
not a drop-in; pick it when ties-to-even is what you actually want. The `l*` variants signal
`FE_INVALID` and return an unspecified value when the result does not fit their integer type;
`rint` and `nearbyint` return a floating-point value, so they cannot overflow at all.

There is a catch, and it is the same bit all over again. Whether `round` is exact depends on which
implementation the linker picks, and on Windows that varies by toolchain. mingw-w64 ships one of
its own, in `libmingwex.a` in older releases and in the `msvcrt`-compatibility archives in newer
ones:

```c
double round (double x)
{
  double res;
  if (x >= 0.0)
    { res = ceil (x); if (res - x > 0.5) res -= 1.0; }
  else
    { res = ceil (-x); if (res + x > 0.5) res -= 1.0; res = -res; }
  return res;
}
```

The distance test needs the very `2^-54` the addition destroyed: for `x = 0.5 - 2^-54` the exact
value of `ceil(x) - x` is `0.5 + 2^-54`, which rounds to exactly `0.5`, so `> 0.5` is false and
the answer stays `1`. Three toolchains on one machine, with `round` called through a function
pointer on a value read at runtime:

| compiler suite | imports | `round` binds to | `round(0x1.fffffffffffffp-2)` |
|---|---|---|---|
| MinGW-w64 GCC 13.2 | `api-ms-win-crt-*` | `libmingwex.a`'s `round.o` | **1** |
| w64devkit GCC 16.1.0 | `msvcrt.dll` | `libmsvcr100.a`'s `round.o` | **1** |
| MinGW-w64 GCC 16.1.0 | `api-ms-win-crt-*` | `ucrtbase.dll` import | 0 |
| `ucrtbase.dll`, via `GetProcAddress` | — | — | 0 |

The two UCRT builds import the same CRT DLLs and still disagree, so the flavour of the toolchain is
not the variable. `round` is a library symbol rather than anything the compiler generates, and the
link is what chose; constant folding is exact in all four, so `round` of a literal prints `0` and
only a value arriving at runtime reaches the library. `rint`, `nearbyint` and `lrint` are `0` on
all four paths, because they are what the hardware computes directly and `roundsd` has no ties-away
mode; building ties-away out of other operations is where a second rounding gets its chance.
gfortran 13.2 from the first toolchain lowers `NINT` for `double` to a call to `lround`, so it
returns `1` as well.

The same archive has done this before: its `__mingw_sscanf`, which an ordinary `-std=gnu23` build
redirects every `scanf` call to, parses `%lf` through 80-bit `long double` and comes back one ULP
low for `0.935469305050944`, which is the bug behind
[One ULP Off](/posts/one-ulp-off-a-floating-point-bug-that). Two weeks, two unrelated functions,
the same library, and the same mistake on my side both times: assuming the name in the source is
the code that runs, when `libmingwex.a` had replaced it.

If the libm call is off limits, add the double just below `0.5` instead of `0.5`:

```c
adj = 0x1.fffffffffffffp-2;      /* nextafter (0.5, 0.0) */
adj = copysign (adj, x);
return x + adj;
```

That is a real glibc implementation of `lround`, the Alpha one, comment and all, and it is also
where Cuoq's post ends, after a detour through a Squeak bug report. It works because the sum can
no longer sit exactly on the tie under an integer. With `0.5`, the input `0.5 - 2^-54` produced
exactly the midpoint `1 - 2^-54`; with the predecessor, it produces `1 - 2^-53`, which is
representable and rounds down. In the `ulp == 1` range the shift does the same job:
`x + 0.5 - 2^-54` is no longer halfway between `x` and `x + 1`. Measured:
`(int)(x + 0x1.fffffep-2f)` is correct for every `float` in `[0, 2^31)`, and
`(long long)(x + 0x1.fffffffffffffp-2)` has no failing window up to `2^63`.

It is still worse than `lround`. The negative side needs `copysign`, it says nothing about
saturation, and one ulp less of threshold is a fragile thing to reason about.

For the positive-only case, with no libm and no wider type, two exact steps also work:

```c
double f = floor(x);                    /* exact */
int    i = (int)f + ((x - f) >= 0.5);   /* x - f is exact and < 1 */
```

`x - floor(x)` is the fractional part and is exactly representable, so the comparison against
`0.5` is exact and the `+ 1` is integer arithmetic. I checked this over every float in
`[0, 2^31)` and over the binade-boundary scan for `double`: zero wrong answers. For
`|x| >= 2^52` the fraction is identically zero and the idiom degenerates to `(int)x`, which is
correct because `x` is an integer there. It still has to be range-checked.

Measured behaviour of the four candidates, for reference:

| expression | `0x1.fffffffffffffp-2` | `-2.5` | semantics |
|---|---|---|---|
| `(int)(x + 0.5)` | **1** | **-2** | half-up, positives only |
| `floor(x) + (x-floor(x) >= 0.5)` | 0 | -2 | half-up |
| `lround(x)` / `round(x)` | 0* | -3 | half-away-from-zero |
| `lrint(x)` / `rint(x)` | 0 | -2 | half-to-even |

`*` glibc, and every Windows toolchain that binds `round` to `ucrtbase.dll`. Some MinGW-w64 builds
link their own; see above.

Other languages have largely reached the same conclusion:

* **Java**: the older `Math.round(double)` was `(long)Math.floor(a + 0.5d)`, i.e. exactly the
  expression under discussion, and this input was exactly what broke it. Modern JDKs do not add
  at all: JDK 8 replaced it with a bit-manipulation routine, with a separate branch for the case
  where `ulp(a) >= 1`, the "integer binade" failure mode described above. On JDK 25,
  `Math.round(x)` is `0`. The raw idiom is still broken in Java, though: `(int)(xf + 0.5f)` is `1`.
* **JavaScript**: the spec now defines `Math.round(x)` as the integral Number closest to `x`,
  ties toward `+∞`; `Math.round(0.49999999999999994)` is `0` in V8.
* **Python**: `round(0.49999999999999994)` is `0`; ties go to even.
* **Fortran**: `NINT` is the standard's nearest-integer function, ties away from zero, and older
  revisions stated the result as `INT(A + 0.5)` for positive `A`. J3 interpretation 000012 (2000)
  settled that such text "specifies the mathematical results, not the computational results", so
  no processor is required to compute it that way; the same document proposes rewording `NINT`
  and `ANINT` as "the integer nearest `A`". J3's own example of the failure was
  `ANINT(16000001.0)` evaluating to `16000002.0`, which is the `ulp == 1` mechanism above. The
  implementation is where the bug returns: gfortran 13.2 on Windows lowers `NINT` for `double` to
  a call to `lround`, so it inherits whatever that does.

All four specify the answer and leave the computation to the implementation. It is also worth
deciding which rounding you mean before writing the code: half-up, half-even and
half-away-from-zero disagree on `0.5`, `2.5`, `-1.5` and `-2.5`, and a test suite built around
one of them will not notice if you implement another. `(int)(x + 0.5)` is a half-up
implementation that is also wrong at one boundary and wrong for most negative inputs.

## Appendix A: the case split in full

> For every finite `double x` with `0 <= x < 2^31 - 0.5` (every input for which the cast to
> `int` is defined and the intended answer is round-half-up), `(int)(x + 0.5)` differs from
> `floor(x + 0.5)` at exactly one value, `0x1.fffffffffffffp-2`, under the default
> round-to-nearest.

The rounding mode is part of the statement: under `FE_UPWARD` the same single input fails and
nothing else does, and under `FE_DOWNWARD` nothing fails at all, since rounding down cannot push
a sum past an integer.

Write `S = x + 0.5` as an exact real, `R = round(S)` for the double the hardware actually
produces, and `n = floor(S)`. The idiom is wrong iff `floor(R) != n`. Since `R` is a neighbour
of `S` and `S >= n`, that requires `R >= n + 1`, which needs

```
S > n + 1 - ulp(S)/2
```

or `S` exactly on that boundary with a tie that rounds upward. It also requires `R != S`, so `S`
must not be representable. There are exactly two ways for that to happen:

**(A) `ulp(x) >= 1`.** Then `x` is an integer and `0.5` is at most half a step, so `S` rounds to
`x` or to `x + 1`: the `ulp == 1` tie that also bites `(long long)`. Inside the `int` range this
cannot happen, because `ulp(x) >= 1` means `x >= 2^52`.

**(B) `S` crosses a binade boundary.** If `S` and `x` share a binade, `S` is a multiple of
`ulp(x)` and so is `0.5`, hence `S` is exact and nothing can go wrong. Crossing means
`x < 2^j <= S` for some `j`, i.e. `x ∈ [2^j - 0.5, 2^j)` while `x` itself lies in
`[2^(j-1), 2^j)`.

In case (B), `S ∈ [2^j, 2^j + 0.5)`. Split on `j`:

* **`j >= 0`**: `2^j` is an integer, so `n = 2^j` and the next integer is `2^j + 1`. Reaching it
  needs `S` within `ulp(S)/2 <= 2^-23` of `2^j + 1`, but `S < 2^j + 0.5` is at least `0.5` away.
* **`j <= -2`**: `S < 2^j + 0.5 <= 0.75`, so `n = 0` and the next integer is `1`. Reaching it
  needs `S > 1 - ulp(S)/2`, far above `0.75`.
* **`j = -1`**: `S ∈ [0.5, 1)`, `n = 0`, and the window is `[1 - 2^-54, 1)`. That window forces
  `x = S - 0.5` into `[0.5 - 2^-54, 0.5)`, which sits inside `[0.25, 0.5)` where the grid is
  `2^-54`. The only representable `x` there is `0.5 - 2^-54`, and it lands exactly on the tie.

The argument never used the width of the significand, only where `2^j` stops being an integer
and how fine the grid is below `0.5`. That is why the same machinery produces a different count
for `float`.

The case analysis is also code. For each of the 1105 binades from the subnormals up to `2^31`,
the candidate window is computed in exact rational arithmetic and intersected with the set of
representable inputs. Not sampled, intersected:

```
binades with a non-empty candidate window: 1
counterexamples: 1
  j= -1  x = 0x1.fffffffffffffp-2  (0.49999999999999994)
```

A 16,384-point scan around every `2^j - 0.5` finds the same single input. Uniform random
sampling over `[0, 2^31)` finds nothing, which is expected and is not evidence: one failure in
4.7 × 10^18 is below the resolution of any practical sampler.

## Appendix B: how this was checked

* **float32**: all 2^32 bit patterns, enumerated in chunks, with the "correct" answer computed
  in `double` (exact for a float plus `0.5`). The counts above are restricted to `[0, 2^31)`,
  where the cast is defined.
* **double**: the range `0 <= x < 2^31 - 0.5` is covered exactly, not by sampling: the same
  enumeration as Appendix A, and it is written with `fractions.Fraction` throughout, so the
  comparison is against exact rationals rather than against hardware.
* **large inputs**: the `ulp == 1` window and both integer limits were checked at the exact
  endpoints (`2^52`, `2^53 - 1`, `2^53`, `INT_MAX`, `INT_MAX + 0.5`). `DBL_MAX + 0.5` was run
  under `fesetround` plus `fetestexcept`:

  ```
  nearest : 0x1.fffffffffffffp+1023  overflow=0 inexact=1
  upward  : inf                      overflow=1 inexact=1
  ```

  `FE_INEXACT` and nothing else under the default mode; under `FE_UPWARD` the exact sum is
  larger than the largest finite double, and IEEE 754 says that overflows.
* **rounding modes**: `(int)(x + 0.5)` was measured under `FE_TONEAREST`, `FE_DOWNWARD` and
  `FE_UPWARD` on the boundary values. `FE_DOWNWARD` flips the offending input to the correct
  answer, and the other two agree.
* **the predecessor fix**: `(int)(x + 0x1.fffffep-2f)` was checked over every float in
  `[0, 2^31)` (zero wrong) and `(long long)(x + 0x1.fffffffffffffp-2)` over the same exact
  window analysis up to `2^63` (no non-empty window).
* **the library functions**: `rint`/`nearbyint`/`lrint` and `round`/`lround` were measured on
  Windows 10.0.26300 with three GCC suites (MinGW-w64 13.2, w64devkit 16.1.0, MinGW-w64 16.1.0),
  using values read at runtime, function pointers so that nothing is inlined, and `GetProcAddress`
  against `ucrtbase.dll`. The four paths disagree, so only the runtime numbers are reported above,
  together with which symbol each link chose.
* Cross-checks against the hardware in C (GCC 13, x86-64, UCRT), Java 25, V8, CPython, gfortran
  13.2.
