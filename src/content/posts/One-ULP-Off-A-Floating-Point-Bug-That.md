---
title: "One ULP Off: A Floating-Point Bug That Wasn't in the C Runtime I Blamed"
author: ntysdd
pubDatetime: 2026-10-07T12:01:57Z
description: |
 Some doubles came back one bit off when parsed, and the culprit was not the 30-year-old CRT I had linked against —
 it was the C99 stdio shim sitting on top of it, which parses floating point through 80-bit long double. A short
 story about double rounding, and about verifying which function you actually call.
tags:
  - Floating-Point
  - C

---


There is a DLL in every 64-bit Windows installation called `msvcrt.dll`. For historical
reasons it smells like "the Microsoft Visual C++ runtime", and for equally historical
reasons a lot of toolchains link against it on purpose. I did that on purpose, and then I
spent half a day chasing a floating-point bug through my own code, through `libm`, and finally
to the CRT.

Except the CRT was innocent. The bug was in the layer *above* it — a layer I did not know
I had enabled.

This post is the long version: where `msvcrt.dll` came from, why it is still around, what
went wrong, and how to check whether your toolchain is doing the same thing to you.

## A very short history of the Windows C runtime

`MSVCRT.DLL` started life in 1995 as the Visual C++ 4.2 runtime that shipped with Windows
95. Every new Visual C++ release meant the Windows team had to pick up the new runtime,
and every fix the Windows team wanted had to be mirrored back into the VC++ runtime. That
did not scale, and it caused real damage: Raymond Chen has [a story about a Y2K fix in
`MSVCRT.DLL` that crashed an application](https://devblogs.microsoft.com/oldnewthing/20140411-00/?p=1273)
because the fixed code used the stack slightly differently and exposed an uninitialized
variable in the application.

There were other problems too. Since every C++ version shared one DLL, binary
compatibility had to be preserved across compiler versions (it was not). Windows 95 and
Windows 98 shipped *different* `MSVCRT.DLL`s that were not compatible with each other.
And since the DLL lived in the system directory, one careless installer could downgrade
it and break the entire OS.

Microsoft eventually gave up and declared `MSVCRT.DLL` an operating-system DLL, off-limits
to applications. Newer Visual C++ releases got versioned runtimes: `msvcr71.dll`,
`msvcr80.dll`, `msvcr90.dll`, `msvcr100.dll`, `msvcr110.dll`, `msvcr120.dll`, one per
compiler, each shipping alongside your program. The one in `System32` stayed frozen in the
past, roughly at the level of the Visual C++ 6.0 C library (C89 plus assorted extensions),
because changing it would break the applications that (against all advice) kept linking it.

Then, with Visual Studio 2015, Microsoft did
[the great CRT refactoring](https://devblogs.microsoft.com/cppblog/the-great-c-runtime-crt-refactoring/):
the runtime was split into

* `vcruntime140.dll`: compiler support (startup code, exception handling, intrinsics); and
* `ucrtbase.dll`, the **Universal CRT** (UCRT), containing the actual C library.

No new DLL per Visual Studio release anymore: the UCRT is serviced in place, and it is a
Windows component. Windows 10 and later ship it; Vista through 8.1 get it through Windows
Update (KB2999226), and applications may also deploy it locally.

The UCRT is a real upgrade, and one part of that upgrade is directly relevant here:
**Microsoft rewrote floating-point formatting and parsing.** The old algorithms were only
accurate to a point. Without going too deep into it, the pre-2015 implementation
considered at most 17 significant decimal digits and discarded the rest. Rick Regan
documented the resulting 1-ULP conversion errors extensively:
[incorrectly rounded conversions](https://www.exploringbinary.com/incorrectly-rounded-conversions-in-visual-c-plus-plus/),
[round-trip failures](https://www.exploringbinary.com/incorrect-round-trip-conversions-in-visual-c-plus-plus/), and
[the new code that was still broken in the VS2015 RC](https://www.exploringbinary.com/visual-c-plus-plus-strtod-still-broken/).
His test program found errors at a rate of "several hundred per million" inputs.

Now, `msvcrt.dll` never got that fix. It is still in `System32`, still doing 1990s
decimal-to-binary conversion. Which means that if you deliberately link against it for
maximum compatibility with old Windows, you inherit those bugs. That, more or less, is the
conventional wisdom, and it is *mostly* true.

## Why I was on `msvcrt.dll` in the first place

I needed binaries that run on old Windows versions without shipping a CRT
redistributable and without an app-local UCRT deployment. The simplest way to get that is
to link against the C runtime that has been part of Windows for 30 years:
`msvcrt.dll`.

In practice, on the MinGW-w64 side that means using an "MSVCRT" toolchain build. I used
[w64devkit](https://github.com/skeeto/w64devkit), which describes itself precisely this
way (*an MSVCRT toolchain*), and produces executables whose only imports are `KERNEL32.dll`
and `msvcrt.dll`:

```
$ objdump -p prog.exe | grep 'DLL Name'
	DLL Name: KERNEL32.dll
	DLL Name: msvcrt.dll
```

That is exactly what I wanted. It is also, as it turned out, only half the story.

## The symptom

Some numbers read from text files were coming out one bit different from the same numbers
read by any other tool on the same machine. Not "slightly different" in a hand-wavy way —
*different doubles*. Values that should have been equal to previously computed values were
not equal, and code downstream that compared them did the wrong thing.

I did the usual tour. I checked my parsing code, my scaling logic, my accumulation. I
suspected `libm`. I checked the FPU control word and the rounding mode. I suspected my own
sanity, in roughly that order, for about half a day.

## The five-minute fuzzer

Eventually I stopped reading code and wrote the cheapest possible oracle: generate a
number, print it so that it round-trips exactly, then parse it back with two different
functions and see if they agree.

```python
import random

for i in range(1, 100000):
    print(random.random())
```

```c
#include <stdio.h>
#include <stdlib.h>

#define BUF_SIZE 1000
char buf[BUF_SIZE];
int main()
{
    while (fgets(buf, BUF_SIZE, stdin))
    {
        double v1;
        sscanf(buf, "%lf", &v1);
        double v2;
        v2 = strtod(buf, NULL);
        if (v1 != v2)
        {
            printf("%s", buf);
        }
    }
}
```

It found the first one almost immediately:

```
0.935469305050944
0x1.def5d52f3618p-1
0x1.def5d52f36181p-1
```

The first hex value is what `sscanf` produced, the second what `strtod` produced. They look
almost identical, and they are one unit in the last place apart: `0x1.def5d52f3618` is
`0x1.def5d52f36180` with a trailing zero dropped, so the last hex digit differs by exactly
`1` — one ULP.

Which one is right? `strtod`:

```
>>> float("0.935469305050944").hex()
'0x1.def5d52f36181p-1'
```

So `sscanf` was low by one ULP. Fine. Case closed: the old CRT's `sscanf` is broken. I
wrote the post in my head and started composing the "beware old CRTs" conclusion.


## Checking my assumption broke it

**In my binary, `sscanf` does not come from `msvcrt.dll`.**

MinGW-w64 enables ANSI stdio compatibility under specific conditions (C99 or later,
C++11 or later, or one of the POSIX/SVID/X-Open/GNU feature macros; the full list is in
`_mingw.h`). Since GCC now defaults to `-std=gnu23`, every ordinary build qualifies; compile
with `-std=c89` and no feature macros and you get the CRT's own stdio back. That decision is
baked into the preprocessor and the link:

```
$ /d/w64devkit/bin/gcc -E -dM -include stdio.h - | grep -E 'ANSI_STDIO|MSVCRT_VERSION'
#define __MSVCRT_VERSION__ 0x600
#define __USE_MINGW_ANSI_STDIO 1

$ nm show.exe | grep sscanf
0000000140002930 T __mingw_sscanf
0000000140002960 T __mingw_vsscanf
```

`__mingw_sscanf` is **defined inside my own executable**, statically linked from
MinGW-w64's `libmingwex`, because `__USE_MINGW_ANSI_STDIO=1` redirects the `scanf` family
(and `printf`) to MinGW's own implementations. `msvcrt.dll` is still imported, for
everything else, but not for this.

Two experiments confirm it:

```
# 1. Turn the shim off, and the disagreement disappears entirely:
$ gcc -D__USE_MINGW_ANSI_STDIO=0 -o check_noansi check.c
$ printf '0.935469305050944\n' | ./check_noansi
$ echo $?
0

# 2. Ask the real System32\msvcrt.dll directly (loaded by hand, since our
#    own sscanf is a different function):
$ cat > probe.c <<'EOF'
    HMODULE h = LoadLibraryA("msvcrt.dll");
    sscanf_fn s = (sscanf_fn)(void *)GetProcAddress(h, "sscanf");
    double v; s("0.935469305050944", "%lf", &v);
    printf("%a\n", v);   /* -> 0x1.def5d52f36181p-1 : CORRECT */
EOF
```

On the exact input that started all of this, the legacy CRT in `System32` returns the
*correctly rounded* value, and MinGW's replacement `sscanf` does not.

So the bug is in MinGW-w64's `__mingw_sscanf`.

## Root cause: everything goes through 80-bit `long double`

Why would a `%lf` conversion get the wrong answer? Because the field is not parsed *as a
`double`*. Look at what the archive says about the object that implements it:

```
$ nm -A libmingwex.a | grep -E 'mingw_sscanf|mingw_sformat'
lib64_libmingwex_a-mingw_sscanf.o  : T __mingw_sscanf
lib64_libmingwex_a-mingw_sformat.o : U __mingw_strtold
```

The scanf implementation pulls in `__mingw_strtold`, the **`long double`** parser. MinGW
on x86-64 uses the x87 80-bit `long double`, so `sscanf("%lf", &d)` computes a 64-bit
mantissa and then narrows it to a 53-bit `double`. That is *double rounding*, and double
rounding is not the same as rounding once.

### This is a known bug, and the report says the same thing

I am not the first person to trip over this. [mingw-w64 bug #989](https://sourceforge.net/p/mingw-w64/bugs/989/)
describes the identical mechanism and even points at the line: the scanner in
`mingw_vfscanf.c` calls `__mingw_strtold` and then rounds to `double`. Its minimal repro is
two lines, and it does not need any exotic input: an eleven-digit literal is enough:

```c
double scanned;
sscanf("1337.1337221", "%lf", &scanned);
/* scanned      = 0x1.4e488ee723902p+10  (1.33713372209999989e+03)
   the same value as a double literal is 0x1.4e488ee723903p+10 (1.33713372210000011e+03) */
```

I verified both of the report's examples on my toolchain; they are 1 ULP off in the expected
direction, and the report's own text asks the obvious question, *why not just use
`__mingw_strtod`?*

Whether this counts as a bug is a separate argument. Codeforces' `testlib` hit the same
problem and [closed its copy of the report](https://github.com/MikeMirzayanov/testlib/issues/203)
with "I don't think guarantees like `scanned == val` are expected when using scanf-like
functions". C's wording for `strtod` requires rounding to the nearest representable value,
with latitude reserved for exact ties, and the `scanf` family is generally read as "as if by
`strtod`", so a 1-ULP-low result on an input that is *not* a tie looks like a deviation
rather than latitude. I will leave that argument to people who enjoy standards law. The
mechanism is what matters here.

I ran the same comparison over a million strings from `repr(random.random())`:

```
inputs                        : 1000000
mingw_sscanf != strtod        : 13      (0.00130%)
mingw_sscanf != (double)strtold : 0     <-- always the long double path
msvcrt_sscanf != strtod       : 0
```

Every disagreement with the correctly rounded `strtod` is exactly a case where
`(double)strtold(...)` disagrees too. Thirteen failures in a million is about 1 in 77,000:
rare enough to survive casual testing, common enough that a 100,000-line fuzz finds one.

### The failure, bit by bit

For `0.935469305050944`:

* The correctly rounded `double` is `0x1.def5d52f36181p-1`.
* The two candidate doubles are `0x1.def5d52f36180p-1` and `0x1.def5d52f36181p-1`, one ULP
  (2^-53) apart, so their midpoint is half an ULP (2^-54) below the correct value.
* The exact decimal value lies just **8.4 × 10⁻⁵ ULP above that midpoint** (about
  9.3 × 10⁻²¹, far below what a 53-bit mantissa can represent).
* Rounding it to an 80-bit `long double` (64-bit mantissa) is *correct*, and it lands
  **exactly on the midpoint**. The value calculated by `strtold` here is
  `0xe.f7aea979b0c04p-4`, which is precisely the tie.
* Narrowing that to `double` now has to break the tie, and the IEEE default is
  round-half-to-even. The even candidate is the one ending in `0`. So the result is
  `0x1.def5d52f36180p-1` — one ULP low.

Nothing in the chain made an arithmetic mistake. The first rounding *destroyed the
information* the second rounding needed. This is why double rounding is such a nasty class
of bug: each step is individually correct.

Sixty-four bits of intermediate precision is not "more than enough to round to 53 bits".
It is enough *in the overwhelming majority of cases*, and wrong in about 1 in 10⁵ of them.

## Meanwhile, the CRT I blamed really is inaccurate

To be clear, the conclusion "old CRT, be careful" is still correct — just not for the
reason I originally thought. When I called `System32\msvcrt.dll`'s own functions directly
on inputs in the range that the legacy algorithm handles badly:

| input | msvcrt `sscanf` | msvcrt `strtod` | correct |
|---|---|---|---|
| `0.935469305050944` | correct | correct | — |
| `6929495644600919.5` | `0x1.89e56ee5e7a57p+52` | same | `…a58p+52` |
| `9214843084008499` | `0x1.05e6cec577619p+53` | same | `…61ap+53` |
| `0.3932922657273` | `0x1.92bb352c46239p-2` | same | `…623ap-2` |

All 1 ULP low, exactly the class of errors Rick Regan reported against pre-2015 MSVC. Two
things follow from this table. First, the legacy CRT is genuinely unreliable for exact
parsing, so if you are on `msvcrt.dll` and you care about the last bit, you should not use
its `sscanf`/`atof`/`strtod`. Second, note that `sscanf` and `strtod` *agree* on the wrong
answer: the entire reason I caught my MinGW case is that in a MinGW binary, the two
functions are implemented by different code, by different people, with different quality.
That accident is a free correctness oracle, and I intend to keep using it.

Also note that on short, 15–17 digit inputs (`repr(random.random())`), `msvcrt.dll` was
correct on all one million samples. The legacy bugs need the right input shape to show up.
A successful fuzz run is not evidence of correctness.

## A note on the code I just picked apart

I want to be explicit about something, because a post like this can easily read as "MinGW
sucks".

That `__mingw_sscanf` exists at all is a service to everyone who compiles C for Windows.
`msvcrt.dll`'s stdio is C89-era: it does not support `%lld` for `long long` (which C99 and
C++11 require), its `vsnprintf` returns `-1` instead of the length that would have been
written, and so on. Rather than making every program on earth work around that, MinGW-w64
reimplements the entire printf/scanf engine, [documents the trade-off](https://sourceforge.net/p/mingw-w64/wiki2/printf%20and%20scanf%20family/),
and gives you `-D__USE_MINGW_ANSI_STDIO=0` if you want the old behavior back. That is a
decade-plus of unglamorous maintenance, mostly unpaid, and it is what makes
standards-conforming C work on Windows at all.

It is also, empirically, *more accurate* than the code it replaces, on exactly the inputs
where accuracy is hard. Compare the two parsers on the vectors that the legacy CRT is
documented to get wrong:

| input | MinGW `sscanf` | `msvcrt.dll` `sscanf` |
|---|---|---|
| `6929495644600919.5` | correct | 1 ULP low |
| `9214843084008499` | correct | 1 ULP low |
| `0.3932922657273` | correct | 1 ULP low |
| `0.500000000000000166533453693773481063544750213623046875` | correct | 1 ULP low |

That is the reverse of what I assumed when I started. The thing I blamed for being the
"legacy path" is the replacement engine, and it has no 17-digit cutoff: it gets right every
case in which the CRT underneath it fails.

And the defect itself is a symptom of a genuinely awkward interface rather than
carelessness. GCC on Windows keeps 80-bit `long double`, a deliberate and good choice for
numerics; `msvcrt.dll` knows only 64-bit `long double`. Any stdio layer that has to bridge
32-, 64- and 80-bit floating point onto a runtime that supports a subset of them will have
corners like this one. Encouragingly, the `__mingw_strtod` that the report asks for is
already in the same library, so the fix is close at hand.

So, in order of who deserves it: thank you first to the MinGW-w64 maintainers. **Kai Tietz**
has been with the project since it was forked off the original MinGW in 2007, and a long
list of other people have kept it going since. Between them they maintain a C99 stdio
implementation and several hundred other pieces, mostly for free, so that the rest of us can
just compile C. Much of that code is older than the project itself, and the people who wrote
it are worth naming: the printf engine (`mingw_pformat.c`) is **Keith Marshall**'s, from the
original MinGW project, and MinGW-w64's `strtod` is a wrapper around **David Gay**'s `dtoa`
(`__strtodg`). The scanf path that bit me is a sibling file, and I am not going to go digging
for whose line that was.

And then [Chris Wellons](https://nullprogram.com/), whose
[w64devkit](https://github.com/skeeto/w64devkit) is a self-contained toolchain you can
unpack anywhere and use to build binaries that run on Windows without shipping anything
alongside them. Its README states plainly that it is an MSVCRT toolchain, which is the kind
of honesty that got me to the bottom of this in half a day instead of a week.

## What I actually take away from this

**"I link against X" does not mean "I call X's functions."** Check it. On Windows:
`objdump -p prog.exe | grep 'DLL Name'` for imports, `nm prog.exe | grep name` to see
whether a symbol is imported or defined locally, and `gcc -E -dM -include stdio.h -` to see
which stdio flavour your headers selected. On my binary, `sscanf` was *mine*, and the CRT
never got a vote.

**Treat 80-bit `long double` as a double-rounding hazard, not as "extra precision".**
`%Lf`, `strtold`, and "store it in `long double` just in case, we can always narrow later"
are the same trap. If a value ultimately has to be a `double`, parsing it as anything wider
can make the result *worse*, not better. (This is also why a `strtof` implemented on top of
a `double` parser is wrong for a small fraction of inputs.)

**Prefer a known-correct parser when the last bit matters.** David Gay's `dtoa`/`strtod`,
Ryu, `fast_float`, or an exact big-integer-based conversion. On modern MSVC, the UCRT
rewrote `printf`/`scanf`/`strtod` specifically because the old ones were not correctly
rounded, and that rewrite is a good reason to prefer UCRT over `msvcrt.dll` when you can
insist on Vista or later. When you cannot (which is why I was here), bring your own parser.
In C++, the `testlib` thread's suggested way out is `std::from_chars`, if you have C++17.
And note that switching the stdio shim off (`-D__USE_MINGW_ANSI_STDIO=0`) does not solve
this: it returns you to the CRT's own parser, which is equally unreliable, just wrong on
different inputs.

**Keep a two-implementation cross-check in your tests.** `sscanf` vs `strtod` on
round-trippable strings found this in under a minute after half a day of reading code. It costs
five lines. Any time two independent implementations of the same conversion exist in your
process, comparing them is nearly free test coverage.

**`repr(random.random())` is a great corpus generator.** Round-trip-printed doubles are
exactly the strings your code will really see, and they exercise the shortest-representation
boundary that naive parsers get wrong. Generate a million of them; they take no effort to
produce and they find these bugs at a useful rate.

I had a correct observation (`sscanf` != `strtod`), a plausible explanation (legacy CRT bug),
and the explanation was wrong. If I had stopped there, I would have published something
confidently useless. Old bugs in old code are real, but "it is probably the old thing" is
not a diagnosis. Load the DLL, call the function, print the hex.

