---
title: "How To Implement a Correctly-rounded powf(x, 3)"
author: ntysdd
pubDatetime: 2026-10-08T08:02:33Z
description: |
 `x * x * x` is wrong for about 8.5% of all positive finite float32 inputs, while there actually is an easy way
 to implement a correctly-rounded one.
tags:
  - Floating-Point
  - C

---

I have been poking at `pow` implementations for a while, and the first thing I found is that
`pow(x, 2)` is not always correctly rounded. That is a strange result to hit early, because
`x * x` is correctly rounded *by definition*: a single IEEE multiplication rounds the exact
product to nearest, so any correctly rounded `pow(x, 2)` has to return exactly `x * x`.

Which raises the next question. `x * x * x` has no comparable guarantee behind it — there is
no single hardware operation to point at and say that it already rounds correctly — so whether
it comes out right is an open question.

The obvious implementation is one line:

```c
float powf3(float x) { return x * x * x; }
```

This is wrong, and it does not take long to find out. Float, not double, because a smaller
format means I can check my work exhaustively instead of arguing about it. Feed `x * x * x`
random floats and compare it with the same expression computed in double:

```c
for (long i = 0; i < 1000000; i++) {
    float x = random_positive_float();
    float a = x * x * x;
    float c = (float)((double)x * x * x);
    if (a != c) printf("%a  %a  %a\n", x, a, c);
}
```

It prints disagreements within microseconds. But "these two differ" only tells you that at
least one of them is wrong; it does not tell you which. So before believing anything, I
computed `x^3` exactly.

That part is easy for a float, and it is the reason this whole exercise is checkable at all.
A float is `M * 2^s` for some integer `M` narrower than 2^24 and some integer `s`, so

```
x^3 = M^3 * 2^(3s)
```

exactly, and `M^3` fits comfortably in 72 bits. So the exact cube of a float is a 72-bit
integer times a power of two, and an arbitrary-precision type can hold it with no rounding at
all — which is easier than writing the integer rounding by hand:

```java
float cube(float x) {
    return new BigDecimal(x).pow(3).floatValue();
}
```

The constructor takes the float's exact binary value, `pow(3)` is computed exactly, and
`floatValue()` does the one rounding that lands back on the float grid. (`BigDecimal` has no
`float` constructor, by the way: the `float` widens to `double` first, which is exact, so the
value still arrives intact.)

With that reference in hand, the verdict on the naive version is not close. Over **all
2,139,095,039 positive finite float32 values**:

| expression | wrong |
|---|---|
| `x * x * x` (all float) | **182,619,688 = 8.54%**, always off by exactly 1 ulp |
| `(float)((double)x * x * x)` | **0** |

Eight percent is not a rare edge case. The smallest float greater than 1 whose cube is wrong
under `x * x * x` is `1.000141f`:

```
x         = 1.000141f
x * x * x = 1.0004231f
correct   = 1.0004232f    <- one ulp up
```

## The version that is unexpectedly right

```c
float powf3(float x) { return (float)((double)x * x * x); }
```

Zero counterexamples in a fuzz, and zero wrong answers in the exhaustive run above. That is
the part that surprised me, because widening to double and narrowing again is exactly the
kind of code that a floating point lecture tells you not to trust. Two roundings should be
worse than one, not better than the naive two-rounding version in float.

To see why it works, count the roundings and look at where they happen.

**Step one: `(double)x * x` is exact.** `x` has 24 significant bits, so `x^2` has at most 48,
and a double holds 53. No rounding happens at all here. This is the load-bearing fact.

**Step two: multiplying that by `x` is a single correctly rounded double operation.** Its
inputs are exact, so the result is exactly `round_53(x^3)`: one rounding, at 53 bits.

**Step three: narrowing to float is a second rounding, but it is harmless.** The condition
for that is known. If you round an exact value first to `q` bits and then to `p` bits, the
result equals the correctly rounded `p`-bit value whenever

```
q >= 2p + 2
```

(round-to-nearest; the condition goes back to Figueroa in 1995 and has been formalized since).
Here `p = 24`, the target is float, and the intermediate is double, so we need `q >= 50` and
we have `q = 53`. Three bits of margin, and the second rounding can never change the answer.

The same inequality explains why the naive float version is wrong, and it explains something
about double that I had not thought about before:

| target | expression | roundings along the way | wrong |
|---|---|---|---|
| float | `x * x * x` | two, at 24 bits | 8.54% |
| float | `(float)((double)x * x * x)` | one at 53 bits, then one at 24 bits: **safe** (53 >= 50) | 0 |
| double | `x * x * x` | two, at 53 bits | 25.6% (sampled) |

That last row is the interesting one. If you are targeting double, `x * x * x` is *not*
correctly rounded, because two roundings at 53 bits are two roundings, and `2p + 2 = 108` is
way out of reach. The reason `(float)((double)x * x * x)` works is not that double is
precise; it is that **float is narrow enough that a double intermediate has room to spare**.
Widening is not a free pass, it is a budget, and the budget depends on the target.

I should be explicit about one caveat, because it is easy to over-generalize this: the
argument assumes round-to-nearest. Under a directed rounding mode (`FE_UPWARD`,
`FE_DOWNWARD`) it does not apply, and I have not checked what the double version does there.

## Verifying it, exhaustively

Float has 2^32 bit patterns. That is small enough to just test all of them, which is a much
better use of an afternoon than trying to write a proof:

```
x^3, exhaustively over 2139095039 positive finite float32 (8.2 s)
  x * x * x            : 182619688 wrong (8.54%), worst 1 ulp
  (float)((double) ... : 0 wrong
  failures span exponents 2^-50 .. 2^42
  first failing x >= 1 : 0x1.00049fp+0 (bits 0x3F80049F)
```

Negative inputs are the mirror image (`(-x)^3 = -(x^3)`, and every operation involved rounds
the magnitude the same way), so the positive sweep plus signs covers all 2^32 finite inputs.

Two disclaimers about trusting that table. First, the exact arithmetic is the easy part; the
step that can go wrong is the final narrowing back to 24 bits, so I cross-checked the result
against a completely independent exact implementation, comparing against the two float
neighbours, over 40,000 sampled inputs
spread across the whole range, including the subnormal and overflow boundaries: zero
disagreements, and the sampled naive failure rate (8.80%) matches the exhaustive one
(8.54%). Second, the naive expression has to
be compiled with contraction off (`-ffp-contract=off`) if you use C, or the compiler may fuse `x * x * x`
into an FMA (or whatever) and you end up measuring a completely different expression.

## What I ended up with

```c
float powf3(float x) { return (float)((double)x * x * x); }
```

My plan when I started was to write something clever: a two-float product to carry the low
bits, an error term, a correcting step at the end. The actual answer is one cast and two
multiplications, and the hard part was not writing it, it was believing it was correct.

Two things I take from this. The `q >= 2p + 2` rule is worth keeping around, because it turns
"widen the intermediate, it's probably fine" into something you can check by counting bits —
and it tells you when the trick *stops* working, which matters more. And at 32 bits wide, the
strongest verification tool is not cleverness but a `for` loop: 2^31 inputs, eight seconds,
and no argument left to have.
