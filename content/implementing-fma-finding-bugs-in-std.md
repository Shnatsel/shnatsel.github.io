+++
title = "Implementing FMA and finding bugs in C and Rust standard libraries"
date = 2026-08-20
description = "This is a story of how I tried to compute `a * b + c`, and found that Rust and musl libc get it wrong."

[taxonomies]
tags = ["Rust", "SIMD", "fearless_simd", "standard library", "musl"]

[extra]
author = 'Sergey "Shnatsel" Davidoff'
+++

This is a story of how I tried to compute `a * b + c`, and found that Rust and musl libc get it subtly wrong.

<!-- more -->

**Fused multiply-add** (FMA) computes `a * b + c` with only one rounding error instead of two. It's an important building block for e.g. trigonometric functions like `sin(x)` and `cos(x)` if you want to implement them accurately.

It's a basic primitive that is usually implemented in hardware, but there is still some hardware out there that doesn't have it. You'd think it would be something like cheap phones, but no, it's Intel. Cheap ARM phones have it and it's been required in 64-bit ARM since the very beginning, but Intel has been launching new parts without AVX2 or fused multiply-add [as recently as 2021](https://en.wikipedia.org/wiki/Tremont_(microarchitecture)).

(I've come to learn that whenever something is holding SIMD back, it's usually Intel.)

15% of machines in the [Firefox hardware survey](https://firefoxgraphics.github.io/telemetry/#view=system) don't have AVX2 and hardware FMA that comes with it, so it has to be emulated for precise algorithms built on top of it to work correctly.

On machines without hardware FMA, Rust's `std::simd` gives up and runs scalar FMA on each `f32` in `[f32; 4]` individually, which is slow. I wanted to do better in [fearless_simd](https://github.com/linebender/fearless_simd) and provide an actually vectorized implementation.

## Emulating FMA with SIMD

Since FMA works on three `f32` values, the total number of possible inputs is 2 to the 96th power. It would take the world's largest supercomputer only 500 years to try them all. We've come a long way! But I need working FMA later this year, so exhaustive verification isn't really on the cards. The best I could do is some known values plus some random tests.

I followed the [2008 paper](https://guillaume.melquiond.fr/doc/08-tc.pdf) "Emulation of FMA and correctly-rounded sums: proved algorithms using rounding to odd" by Sylvie Boldo and Guillaume Melquiond, which has a formal proof of correctness in Coq. That way I don't have to worry about trying to verify the algorithm myself.

For `f32`, their algorithm is refreshingly simple: compute `a * b + c` in `f64`, then round it to `f32`. The only caveat is special handling of rounding errors in the conversion, in case the result falls exactly between two representable values. 

Translating the algorithm to SIMD was also straightforward: just do all that basic math per-lane. The special handling of rounding is needed very rarely - you hit it less than once in a million when processing values in the [-1, 1) range, so just put it under an `if` and it'll be fine. [Done!](https://github.com/linebender/fearless_simd/pull/323)

Benchmarks look great: it's 5x faster than `std::simd`, even bigger than the expected 4x speedup because Rust standard library has to check if FMA is available on the system in every call, and also has to worry about setting floating-point exception flags, both of which add overhead.

## Follow the White Rabbit

Just a few hours later, a wild [**@awxkee**](https://github.com/awxkee) appeared and [posted some inputs](https://github.com/linebender/fearless_simd/pull/323#issuecomment-5233682925) on which my implementation diverged from the hardware results.

Moments like these are why I love open source. I have no idea where he came from or how he even found this PR, he's never contributed code to Fearless SIMD before. But there it was, a counter-example that broke my translation of a formally verified algorithm.

Turns out I translated the paper into code incorrectly. I forgot to add special handling for values very close to zero, called **subnormal** values, which have a slightly different representation, and many of the usual floating-point "tricks" don't work on them. The check that would apply the rounding fixup handled them incorrectly.

> **Aside:** Subnormals are fascinating! Learn all about them [here](https://numerical-rust-cpu-81b2c3.pages.in2p3.fr/19-subnormal-entertainment.html).

So I looked at the paper more carefully and fixed my implementation to match it more closely. Then I added tests covering the problematic inputs, random tests that try a million subnormals, and random tests that try a million values that should require fixup, for good measure.

Wait... Why are tests still failing, but _only_ on old systems?

## Down the rabbit hole

Fearless SIMD has CI configured to run tests in an emulator for every supported SIMD level, on top of running them on the host. This is the only way to test AVX-512 codepaths on CI. It would also fail if we tried to use instructions not available in hardware, although the Rust compiler [already verifies this for us](https://shnatsel.github.io/safe-simd-in-rust-even-on-the-inside/). It's just good practice to test the codepaths on the hardware where they'd actually run.

The built-in `f32::mul_add` in the Rust standard library **has the exact same bug!**

It fails to properly handle rounding for subnormals. As does `std::simd`. But the software implementation is only invoked for systems that don't have FMA in hardware, so without the emulator we wouldn't have noticed.

So I [report the bug](https://github.com/rust-lang/compiler-builtins/issues/1262) to the Rust standard library, then transcribe the formally verified algorithm from [the paper](https://guillaume.melquiond.fr/doc/08-tc.pdf) again (hopefully correctly) to insulate the scalar fallback inside `fearless_simd` from it. Tests pass.

Here's the algorithm, it's not that scary:

```rust
fn scalar_mul_add_precise_f32(a: f32, b: f32, c: f32) -> f32 {
    let product = (a as f64) * (b as f64);
    let c = c as f64;
    let mut sum = product + c;
    if sum.is_finite() {
        let virtual_sum = sum - product;
        let rounding_error = (product - (sum - virtual_sum)) + (c - virtual_sum);
        let sum_bits = sum.to_bits();
        if rounding_error != 0.0 && sum_bits & 1 == 0 {
            let corrected_bits = if sum.is_sign_negative() == rounding_error.is_sign_negative() {
                sum_bits.wrapping_add(1)
            } else {
                sum_bits.wrapping_sub(1)
            };
            sum = f64::from_bits(corrected_bits);
        }
    }
    sum as f32
}
```

Just the Rust standard library left to fix.

## How deep does this go?

Fixing the standard library is trickier because it not only computes the result but also sets the [floating-point status flags](https://en.wikipedia.org/wiki/FLAGS_register). It also has this codepath not just for `f32` using `f64` but also for `f64` using `f128` on hardware where that's available.

So I write tests, for `f32` and `f64` both, then [fix the code](https://github.com/rust-lang/compiler-builtins/pull/1270) to the best of my understanding.

Wait, what's this at the top of the file?

```rust
/* origin: musl src/math/fmaf.c Ported to generic Rust algorithm in 2025, TG. */
```

I wonder if...

```c
if ((u.i & 0x1fffffff) != 0x10000000) /* not a halfway case */
```

Yep, that's the same bug: this check ignores subnormals.

So it's not just Rust std that's buggy, **musl libc is buggy too**. And [they credit this implementation to FreeBSD](https://git.musl-libc.org/cgit/musl/tree/src/math/fmaf.c?id=f21a96538f78fa8e2040831b4209b35f2fb581da), with copyright years 2005-2011, so who knows where else this buggy code was copy-pasted over the past two decades.

Musl has no bug tracker, so let's [report it](https://www.openwall.com/lists/musl/2026/08/10/1) on the mailing list, and pray that it isn't an intentional "optimization" (the mailing list has no search function so I can't check) and that it won't just be forgotten forever once it stops showing up on the recent messages page. Boy do I love development processes from 30 years ago!

## Where is the bottom?!

Oh hey, CI checks for my standard library PR are complete!

Wait, why are they failing, but only on 32-bit ARM? With completely garbage values?!

Somehow the experimental, nightly-only `f128` type gets enabled on 32-bit ARM platforms, and... goes completely haywire? I sure am not going to read that much ARM assembly, but an LLM can [make quick work of it](https://github.com/rust-lang/compiler-builtins/pull/1270#issuecomment-5246545522), and... it's a rustc ABI bug. On this type, on this architecture, localized entirely to my pull request.

> **Aside:** In case you're wondering, using LLMs for analysis is [permitted](https://blog.rust-lang.org/inside-rust/2026/08/05/rust-langrust-is-adopting-an-llm-policy/) by the Rust LLM policy, under certain conditions.

So there's now [a bug report about this too](https://github.com/rust-lang/compiler-builtins/issues/1271) from a maintainer who could verify the LLM findings and understands this area way better than I do.

## Fixes

`fearless_simd` is easy. It's fixed. You can use that and get FMA without this bug.

The nightly-only `f128` bug in the Rust compiler is also fixed.

[My pull request for the Rust standard library](https://github.com/rust-lang/compiler-builtins/pull/1270) broke setting floating-point exception flags in some cases. It occurred to me to run an LLM to search for bugs I could have introduced, and it found a counter-example. So I've put some more work into making a patch for it that touched as little code as possible. It's still awaiting review.

To my surprise, musl developers took my bug report seriously. Both a minimal fix and a substantial rewrite of the `fmaf()` function were quickly proposed and iterated upon based on feedback from other developers. And after all the feedback was addressed... nothing happened. It's still not merged as I'm writing this.

Looks like code review is a bottleneck regardless of the age of your development tools.

## Does this bug matter?

I have no idea.

On one hand, it's just an incorrect rounding. The result is off by one least significant bit in some rare cases. The original counter-example is off by about 15 parts per million of the correct result.

On the other hand, the error might get amplified dramatically depending on what you do with the result. And those math routines with exact error bounds are used for _something_, so whatever that is will get the wrong error bounds.

Perhaps most noticeably, deterministic simulations running across different machines will diverge.

## Buggy FMA? In MY computer?

On x86 this is only a problem on cheap and/or very old Intel without AVX2, and on Chinese Hygon x86 chips that don't have FMA at all. 64-bit ARM is unaffected.

However, this will probably haunt 32-bit ARM and various embedded systems for years to come. I would not be surprised if the buggy FreeBSD implementation got copy-pasted into lots of different toolchains for various obscure platforms.

If FMA accuracy matters to you, check that `fmaf(a, b, c)` with these input bit patterns

```rust
a = 0x97000800
b = 0x1cfff001
c = 0x00010002
```

evaluates to bit pattern `0x00010001` (correct) rather than `0x00010002` (buggy).

