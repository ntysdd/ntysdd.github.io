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
product to nearest, so any correctly rounded `pow(x, 2)` has to return exactly `x * x`. On the
libm I had in front of me (MSVC's UCRT, reached through CPython's `math.pow`)
disagrees with `x * x` for about 1 in 1500 double inputs. (`pow(x, 2)` and `pow(x, 4)` are
their own story and get their own post.)

That is not a complaint about libm authors, by the way. `pow(x, y)` has to be right for every
`y`, correctly rounding it for an arbitrary exponent is far more expensive than a couple of
multiplications, and special-casing small integer exponents is a judgement call about where
to spend that budget. I only get away with it because I get to pick the exponent.

While I was poking, I also looked at `x^3`, and then I did the obvious thing: if the library
cannot be trusted with a fixed exponent, write the function myself. Float, not double,
because a smaller format means I can check my work exhaustively rather than argue about it.

```c
float powf3(float x) { return x * x * x; }
```

This is wrong, and it does not take long to find out. Feed it random floats and compare
against a reference:

```c
/* naivest possible checker; x is a random float, built from a random bit pattern */
for (long i = 0; i < 1000000; i++) {
    seed = seed * 1103515245u + 12345u;
    uint32_t b = (seed >> 1) % 0x7F7FFFFFu;      /* random positive finite float */
    float x; memcpy(&x, &b, 4);
    float a = x * x * x;
    float c = (float)((double)x * x * x);
    if (a != c) printf("0x%08X  %a  %a\n", b, a, c);
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

exactly, and `M^3` fits comfortably in 72 bits. Compute that integer, round it once to the
float grid, and you have the correctly rounded answer by construction:

```c
/* x = M * 2^s exactly, x^3 = M^3 * 2^(3s) exactly; M^3 fits in __int128 */
unsigned __int128 N = (unsigned __int128)M * M * M;
int e2 = bitlen(N) - 1 + 3*s;          /* exact value is in [2^e2, 2^(e2+1)) */
/* round N to 24 significant bits, half-to-even, then assemble the float */
```

With that reference in hand, the verdict on the naive version is not close. Over **all
2,139,095,039 positive finite float32 values**:

| expression | wrong |
|---|---|
| `x * x * x` (all float) | **182,619,688 = 8.54%**, always off by exactly 1 ulp |
| `(float)((double)x * x * x)` | **0** |

Eight percent is not a rare edge case. The smallest float greater than 1 whose cube is wrong
under `x * x * x` is `0x1.00049fp+0`:

```
x        = 0x3F80049F   1.0001410245895386
x^3 exact                1.000423133435225
x * x * x = 0x3F800DDD   1.0004230737686157
correct   = 0x3F800DDE   1.0004231929779053     <- one ulp up
```

The exact cube sits 0.001 ulp above the midpoint between those two neighbours, so the correct
answer is the higher one, and it is barely higher. The exact value is so close to the tie that
any error in either multiplication is enough to fall on the wrong side of it.

Failures start in the subnormal region (`x` around 2^-50, where the cube underflows) and run
all the way up to 2^42, so this is not a "tiny values only" problem. And they are always
exactly 1 ulp: awkward, never catastrophic.

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

Two disclaimers about trusting that table. First, the reference implementation is mine, so it
can be wrong in the same direction as the thing it is checking. I cross-checked it against a
completely independent exact reference — arbitrary-precision rationals in Python, comparing
against the two float neighbours — over 40,000 sampled inputs spread across the whole range,
including the subnormal and overflow boundaries: zero disagreements, and the sampled naive
failure rate (8.80%) matches the exhaustive one (8.54%). Second, the naive expression has to
be compiled with contraction off (`-ffp-contract=off`), or the compiler may fuse `x * x * x`
into an FMA and you end up measuring a completely different expression.

It is also worth saying that the fuzzer was not how I found the bug. A million random floats
find a counterexample almost immediately, but "I generated a million and they agreed" is not
evidence, and "I generated a million and 8.5% disagreed" does not tell you which side is
right. The integer reference is what turned an observation into a fact.

## What I ended up with

```c
float powf3(float x) { return (float)((double)x * x * x); }
```

My plan when I started was to write something clever: a two-float product to carry the low
bits, an error term, a correcting step at the end. The actual answer is one cast and two
multiplications, and the hard part was not writing it, it was believing it was allowed.

Two things I take from this. The `q >= 2p + 2` rule is worth keeping around, because it turns
"widen the intermediate, it's probably fine" into something you can check by counting bits —
and it tells you when the trick *stops* working, which matters more. And at 32 bits wide, the
strongest verification tool is not cleverness but a `for` loop: 2^31 inputs, eight seconds,
and no argument left to have.

`pow(x, 2)` and `pow(x, 4)` behave differently enough that they need their own post. `x * x`
is correctly rounded for free, and `x^4` is where the free lunch starts to depend on the
order you multiply in. That is next.
