---
title: "Two NaNs, One Expression: The Bits Nobody Promised"
author: ntysdd
pubDatetime: 2026-11-23T16:41:00Z
description: |
 `0.0 / 0.0` gives one NaN when `javac` folds it and a different one when the CPU divides it, on the
 same JVM. The JVM spec leaves the bit pattern to the hardware and IEEE 754 does not specify it either,
 so my port now only promises the finite results.
tags:
  - Floating-Point
  - Java
  - C
---

Recently I ported CORE-MATH to Java, with the rule written down before the code: every result must be
bit-identical to the C result. By that I did not mean "correctly rounded on its own terms"; I meant the
same bits, compared with `Double.doubleToRawLongBits`, with any difference counted as a failure. For a
library whose entire point is correctly rounded mathematical functions, that looked like the only
honest definition available.

Implementing against it was mostly uneventful. The failures were tidy: both results were NaN, and their
bits differed. No finite value was wrong, nothing was one ulp off. Two implementations agreed that the
answer was not a number and disagreed about which not-a-number it was.

My first guess was that I had mistranslated one of CORE-MATH's NaN-producing paths, something like
`0.0 / 0.0`, and was looking at my own bug through too many layers. It was not my bug. The first thing
that told me so had nothing to do with CORE-MATH.

## Two NaNs from one line

Four lines of Java, no library involved:

```java
final double folded = 0.0 / 0.0;
double z = 0.0;
System.out.printf("%016x%n", Double.doubleToRawLongBits(folded));
System.out.printf("%016x%n", Double.doubleToRawLongBits(z / 0.0));
```

On JDK 27.0.1, x86-64:

```text
7ff8000000000000
fff8000000000000
```

Same expression, same JVM, same machine, different sign bit. One is evaluated at compile time, the
other at run time, and `-Xint` prints the same two lines, so it is not the JIT. The difference comes
from `javac`, which folds the first one into a constant.

The reason is short. When the hardware performs an invalid operation like `0.0 / 0.0`, it returns its
default quiet NaN, `0xfff8000000000000`, with the sign bit set and a zero payload. When `javac` folds
the same division, the division never happens: the value is computed by the compiler and stored in the
class file as `0x7ff8000000000000`, which is `Double.NaN`.

## One NaN value, many bit patterns

[JVMS §2.3.2](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-2.html#jvms-2.3.2) keeps the
two halves apart. IEEE 754 allows many distinct NaN values; the Java platform treats them as collapsed
into a single canonical one. And then, a few paragraphs later:

> IEEE 754 specifies a set of NaN bit patterns, but does not mandate which particular NaN bit pattern
> is used to represent a NaN result; this is left to the hardware architecture.

That last clause is the one I should have read before writing the harness. The language has a single
NaN value, and which bit pattern carries it is left to whatever produced the value.

The API keeps the same two questions apart: `floatToIntBits` canonicalizes, `floatToRawIntBits` does
not. My harness wanted to know whether the value was the same and reached for the second one.

Both patterns were legitimately NaN, and neither my port nor `javac` had done anything wrong. That let
me relax a little.

## What the standard actually promises

If the bits are left to the hardware, the interesting question is what the other side of my comparison
was promising. IEEE 754-2019 separates two cases that the harness had merged:

- A NaN **input** propagates its payload, with a signaling NaN quieted.
- An invalid operation on ordinary numbers, such as zero divided by zero or the square root of a
  negative, produces a NaN whose payload and sign are unspecified.

My failures were of the second kind: `0.0 / 0.0` has no NaN input whose payload could be propagated, so
the standard says nothing about which NaN comes out. Even the first case is less settled than it looks;
the working group's own
[background note on payload propagation](https://grouper.ieee.org/groups/msc/ANSI_IEEE-Std-754-2019/background/nan-propagation.pdf)
describes it as "uncertain and not fully standardized."

Comparing those bits therefore tests whether two implementations happened to make the same arbitrary
choice, which is not the same thing as testing correctness.

## Then the reference implementation rewrote its cosine

I could still have argued the other way: CORE-MATH is a fixed body of code, and "bit-identical to this
build" is a well-defined target even if no standard requires it.

In October 2026 CORE-MATH replaced its binary64 cosine implementation with a full rewrite, which is
also faster. No finite result changed; old and new agree bit for bit on the project's own test vectors
and on random inputs. `cos(+infinity)` on x86 did move, from `0xfff8000000000000` to
`0x7ff8000000000000`.

The old code answered an infinite input with `0.0 / 0.0`, so the NaN came from the hardware: negative
on x86-64, positive on AArch64. The new code copies the argument and sets the quiet bit instead, so the
sign follows the argument and no machine gets a vote.

So the rewrite did not change what the cosine function computes. It replaced a place where the code had
been asking the CPU for a NaN with a place where it builds one itself.

That is where the harness stopped making sense to me. If the thing I am being bit-exact *to* asks the
CPU for a NaN, then what I was measuring was one machine's choice, not the library's.

## It is not even stable per machine

The AArch64 half of that has a history. In July 2019 someone asked
[jdk-dev](https://mail.openjdk.org/pipermail/jdk-dev/2019-July/003164.html) why
`floatToRawIntBits(0.0f / 0.0f)` gives a different answer on x86_64 and aarch64. The question came out
of a Spark test that had asserted `2143289344`, the canonical positive NaN. Aleksey Shipilev's reply
was a question of his own: why should it be the same?

So even inside one body of bytecode, on one JDK version, the answer depends on the CPU. A port that
imitated the bits of one C build on one platform would pass there and fail on the next, and neither
outcome would have anything to do with a promise.

## What I changed

I changed what the project promises. It now promises that results agree with the reference for non-NaN
values, and says nothing about which NaN comes back. The harness compares finite results bit for bit,
and where a NaN is expected it only asserts `Double.isNaN`. The README says it in one line: "Which NaN
comes back is not part of the promise."

## What I take away

**`raw` is a different question from equality.** The JVM collapses the NaNs into one; the machine does
not. Every time I reach for the `raw` variant of a conversion, I should know which of those two
questions I am asking.

**A test can be stricter than the contract it checks, and that is a bug in the test.** It fails on
differences that mean nothing, and going green teaches you nothing extra.

The finite results still match, bit for bit. The NaN bits were never something I could have guaranteed,
and I have stopped worrying about them.
