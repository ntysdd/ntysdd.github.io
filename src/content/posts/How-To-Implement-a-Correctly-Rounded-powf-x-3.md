---
title: "How To Implement a Correctly-rounded powf(x, 3)"
author: ntysdd
pubDatetime: 2026-10-08T08:02:33Z
modDatetime: 2026-10-08T10:01:14Z
description: |
 `x * x * x` is wrong for about 8.5% of all positive finite float32 inputs, while there actually is an easy way
 to implement a correctly-rounded one.
tags:
  - Floating-Point
  - C
  - Java

---

I have been poking at `pow` implementations for a while, and the first thing I found is that
`pow(x, 2)` is not always correctly rounded. That is a strange result to hit early, because
`x * x` is correctly rounded *by definition*: a single IEEE multiplication rounds the exact
product to nearest, so any correctly rounded `pow(x, 2)` has to return exactly `x * x`.

`x * x * x` has no comparable guarantee behind it: there is no single hardware operation to
point at and say that it already rounds correctly, so whether it comes out right is an open
question.

The obvious implementation is one line:

```c
float powf3(float x) { return x * x * x; }
```

Unsurprisingly it is wrong, and it does not take long to find out. Float, not double, because a smaller
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

That part is easy for a float: any arbitrary-precision type can hold a float, and its cube,
exactly. Java's `BigDecimal` does this in one line:

```java
float cube(float x) {
    return new BigDecimal(x).pow(3).floatValue();
}
```

The constructor takes the float's exact binary value, `pow(3)` is computed exactly, and
`floatValue()` does the one rounding that lands back on the float grid. (`BigDecimal` has no
`float` constructor, by the way: the `float` widens to `double` first, which is exact, so the
value still arrives intact.) In practice `floatValue()` is correctly rounded.

With that reference in hand, I ran both expressions over every positive finite float32 value:

| expression | wrong |
|---|---|
| `x * x * x` (all float) | **8.54%**, always off by exactly 1 ulp |
| `(float)((double)x * x * x)` | **0** |

The smallest float greater than 1 whose cube is wrong under `x * x * x` is `1.000141f`:

```
x         = 1.000141f
x * x * x = 1.0004231f
correct   = 1.0004232f    <- one ulp up
```

This is the tradeoff behind a rewrite compilers do when the call is marked fast: `pow(x, 3)`
becomes `x * x * x`, and `(x*x)*x` has an extra rounding step. It came up on
[llvm-dev](https://lists.llvm.org/pipermail/llvm-dev/2017-January/108942.html) in 2017, where
Steve Canon measured, for single-precision `x` in `[1, 2)`, a worst-case relative error of 1.29 ulp for
`x*x*x` against 0.500013 ulp for `powf(x, 3)`.

## The version that is unexpectedly right

```c
float powf3(float x) { return (float)((double)x * x * x); }
```

Zero counterexamples in a fuzz, and zero wrong answers in the exhaustive run above. I did not
expect that: widening to double and narrowing again is exactly the kind of code that a
floating point lecture tells you not to trust. Double rounding is the hazard that lecture is
warning about, so it is not where you would expect the fix to be.

To see why it works, count the roundings and look at where they happen.

**Step one: `(double)x * x` is exact.** `x` has 24 significant bits, so `x^2` has at most 48,
and a double holds 53. No rounding happens at all here.

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

The same inequality explains why the naive float version is wrong, and it has something to say
about double as well:

| target | expression | roundings along the way | wrong |
|---|---|---|---|
| float | `x * x * x` | two, at 24 bits | 8.54% |
| float | `(float)((double)x * x * x)` | one at 53 bits, then one at 24 bits: **safe** (53 >= 50) | 0 |
| double | `x * x * x` | two, at 53 bits | 25.7% (sampled) |

That last row is the interesting one. If you are targeting double, `x * x * x` is *not*
correctly rounded.

One caveat: the argument assumes round-to-nearest. Under a directed rounding mode (`FE_UPWARD`,
`FE_DOWNWARD`) it does not apply, and I have not checked what the double version does there.

## Verifying it, exhaustively

Float has 2^32 bit patterns, so it is easier to test all of them than to write a proof:

```
Searching ...
count=2139095039
END
```

Negative inputs are the mirror image (`(-x)^3 = -(x^3)`, and every operation involved rounds
the magnitude the same way), so checking the positive ones is enough.

## What I ended up with

```c
float powf3(float x) { return (float)((double)x * x * x); }
```

My plan when I started was to write something clever: a two-float product to carry the low
bits, an error term, a correcting step at the end. The actual answer is one cast and two
multiplications; believing it was correct took longer than writing it.

The `q >= 2p + 2` rule is worth keeping around, because it turns "widen the intermediate,
it's probably fine" into something you can check by counting bits — and it tells you when the
trick *stops* working. And at 32 bits wide, the cheapest proof is a `for` loop: 2^31 inputs,
and no argument left to have.
