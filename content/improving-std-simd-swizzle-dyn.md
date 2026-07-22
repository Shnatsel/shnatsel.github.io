+++
title = "Improving std::simd::swizzle_dyn"
date = 2026-06-20
description = "Making `std::simd::swizzle_dyn` up to 6x faster, and giving up on actually using it."

[taxonomies]
tags = ["Rust", "SIMD", "optimization", "standard library"]

[extra]
author = 'Sergey "Shnatsel" Davidoff'
+++

[Fearless SIMD](https://github.com/linebender/fearless_simd) recently received a [pull request](https://github.com/linebender/fearless_simd/pull/265) that started like this:

> Swizzles are a whole can of worm and I do not intend to figure out how we should implement this generically right now. 😄
<!-- more -->
The pull request author only wanted to convert RGBA image layouts, e.g. RGBA <-> BGRA, so we solved the immediate need by implementing a [simpler version](https://github.com/linebender/fearless_simd/pull/266) that shuffles bytes within 128-bit blocks. That way we can trivially support this operation even on 512-bit vectors on 128-bit hardware, and 512-bit hardware still gets to utilize its full potential with native instructions.

But that made me wonder: how does [`std::simd`](https://doc.rust-lang.org/std/simd/index.html) implement arbitrary swizzles that aren't confined to 128-bit blocks? Turns out the answer is: **poorly.**

Over the course of this article we're going to fix some of that.

## What is a swizzle?

A swizzle, or a shuffle, rearranges elements within an array.

For example, if I have the input array `[A,D,F,I,M,R,S,T]` and apply the swizzle mask `[2,0,6,7,6,3,4,1]`, I get `[F,A,S,T,S,I,M,D]`.

The `dyn` part indicates that the swizzle mask is dynamic (not hardcoded), so you could supply the swizzle mask `[6,4,0,5,7,0,6,6]` at runtime to get a different word.

This example uses only 8 bytes, or 64 bits. In practice hardware implements shuffles on vector sizes from 128 to 512 bits, and so does `std::simd`.

## Understanding the implementation

On a high level, [the current implementation](https://github.com/rust-lang/rust/blob/d527bc9bfa297ca7fd7f5ae93781eeec42073170/library/portable-simd/crates/core_simd/src/swizzle_dyn.rs) of `std::simd::swizzle_dyn` is very simple: use the native hardware operation if available, otherwise give up and move bytes one by one.

Even when hardware shuffles for a given size are available, `swizzle_dyn` doesn't always map to the hardware cleanly. It promises that the values for out-of-bounds indices will be set to `0`, while x86 hardware shuffle just lets them wrap, so `swizzle_dyn` has to do extra work to uphold this guarantee.

Wait a minute... what is this warning in its documentation?

> Note that the current implementation is selected during build-time of the standard library, so `cargo build -Zbuild-std` may be necessary to unlock better performance, especially for larger vectors. A planned compiler improvement will enable using `#[target_feature]` instead.

Uh oh.

## The black sheep of std::simd

Rust uses [LLVM](https://en.wikipedia.org/wiki/LLVM) for optimizing your code and turning into something CPUs can execute. And LLVM doesn't like dealing with platform-specific operations. It's much easier to implement an operation like "add two SIMD vectors" generically, run all optimizations on this generic form, and then select the appropriate instructions for a given CPU later.

Most `std::simd` operations simply emit the generic form such as "add two SIMD vectors", which makes the implementation remarkably straightforward. 

But [swizzle_dyn](https://doc.rust-lang.org/std/simd/struct.Simd.html#method.swizzle_dyn) does not. And [the implementation](https://github.com/rust-lang/rust/blob/d527bc9bfa297ca7fd7f5ae93781eeec42073170/library/portable-simd/crates/core_simd/src/swizzle_dyn.rs) looks something like this:

```rust
#[cfg(target_feature = "neon"))]
unsafe fn armv7_neon_swizzle_u8x16(bytes: Simd<u8, 16>, idxs: Simd<u8, 16>) -> Simd<u8, 16> {
    use core::arch::arm::{uint8x8x2_t, vcombine_u8, vget_high_u8, vget_low_u8, vtbl2_u8};
    unsafe {
        let bytes = uint8x8x2_t(vget_low_u8(bytes.into()), vget_high_u8(bytes.into()));
        let lo = vtbl2_u8(bytes, vget_low_u8(idxs.into()));
        let hi = vtbl2_u8(bytes, vget_high_u8(idxs.into()));
        vcombine_u8(lo, hi).into()
    }
}
```

I've seen that before! That's no LLVM magic, even if it's completely inscrutable. Those are plain old SIMD intrinsics!

The problem is those platform-specific intrinsics are under `#[cfg(target_feature = ...))]`. It's that `cfg` that kills performance, because it gets resolved very early in the build process, before the context from which the function is called is considered. `fearless_simd` and every other SIMD crate that isn't called [`wide`](https://crates.io/crates/wide) has a [solution](https://shnatsel.github.io/safe-simd-in-rust-even-on-the-inside/) for this, but `std::simd` doesn't, and it can't just adopt the ecosystem solution without a large change to the API.

The warning in documentation doesn't convey the full extent of the problem. Not only is `-Z build-std` necessary, you also have to pass `RUSTFLAGS=-C target-cpu=` to get decent performance. But if you do that and you run the program on a CPU that's older than the one you selected, the program will crash. The typical solution is selecting the best implementation at runtime via crates such as [`multiversion`](https://crates.io/crates/multiversion), but that does not work for `swizzle_dyn`, even with `-Z build-std`.

And let me be clear: I'm not a compiler engineer. I'm not the one who's going to build whatever compiler improvement is planned to fix this.

But I am a `fearless_simd` maintainer, and I already know more about SIMD intrinsics than ever wanted to. So there is plenty here that I can improve.

## No NEON?

The first thing that stuck out to me when reading the [swizzle_dyn code](https://github.com/rust-lang/rust/blob/d527bc9bfa297ca7fd7f5ae93781eeec42073170/library/portable-simd/crates/core_simd/src/swizzle_dyn.rs) is that it only implements 128-bit swizzles on ARM NEON, even though NEON does have hardware instructions for larger swizzles.

It turned out that `std::simd` isn't actually developed as part of the Rust standard library - it has [its own repository](https://github.com/rust-lang/portable-simd/) where development happens, and the result is periodically synced into the standard library. And the missing NEON implementations were [already implemented there](https://github.com/rust-lang/portable-simd/blob/9acfa97d5ce89df77c172b3d6942cafe688c18da/crates/core_simd/src/swizzle_dyn.rs#L118-L183).

That wasn't at all apparent when browsing the code. I nearly wasted a lot of time making pull requests for an outdated version of the code in the wrong repository. Adding headers to synced-in files that specify how to contribute to them would save other people in my position a lot of time.

## Doubling the vector width

Let's say we want to execute `[A,D,F,I,M,R,S,T].swizzle_dyn([2,0,6,7,6,3,4,1])` but all we have is half-width vectors. `swizzle_dyn` setting out-of-bounds elements to zero is going to help us here.

To process the first half, `[2,0,6,7]`, we run `[A,D,F,I].swizzle_dyn([2,0,6,7])` and `[M,R,S,T].swizzle_dyn([-2,-4,2,3])`, which gives us `[F,A,0,0]` and `[0,0,S,T]`. Then we combine these two intermediate results with a cheap bitwise OR, et voilà! We get `[F,A,S,T]`! Now repeat the same for the second half, and we're done!

It is rare that a portability guarantee actually helps optimize something instead of incurring additional work, but this time we're in luck!

This takes four shuffles instead of one, but it's still much faster than moving bytes around one at a time. I've measured a 4x performance improvement from implementing this on AVX2 for 512-bit vectors inside `fearless_simd`.

This is now [merged](https://github.com/rust-lang/portable-simd/pull/540) into `std::simd`, and will be available in the next release of `fearless_simd` too.

Using this approach for 4x the vector size would require 16 shuffles, so I don't think it's worth it, but it might still be worth benchmarking.

## Optimizing AVX2

AVX2 is weird. It has 256-bit vectors, but most operations process 128-bit halves independently. That's fine for most operations, but problematic for a swizzle that may move values across those halves. So the AVX2 implementation needs two separate steps: shuffling within 128-bit blocks, and then using a trick similar to what we've just done.

Another complication is that `swizzle_dyn` promises that values for out-of-bounds indices will be set to `0`, which x86 hardware shuffles don't do, so we have to implement that in software.

The [current implementation](https://github.com/rust-lang/portable-simd/blob/9acfa97d5ce89df77c172b3d6942cafe688c18da/crates/core_simd/src/swizzle_dyn.rs#L206-L223) looks like this:

```rust
use x86::_mm256_permute2x128_si256 as avx2_cross_shuffle;
use x86::_mm256_shuffle_epi8 as avx2_half_pshufb;

let mid = Simd::splat(16u8);
let high = mid + mid;

// This is ordering sensitive, and LLVM will order these how you put them.
// Most AVX2 impls use ~5 "ports", and only 1 or 2 are capable of permutes.
// But the "compose" step will lower to ops that can also use at least 1 other port.
// So this tries to break up permutes so composition flows through "open" ports.
// Comparative benches should be done on multiple AVX2 CPUs before reordering this

let hihi = avx2_cross_shuffle::<0x11>(bytes.into(), bytes.into());
let hi_shuf = Simd::from(avx2_half_pshufb(
    hihi,        // duplicate the vector's top half
    idxs.into(), // so that using only 4 bits of an index still picks bytes 16-31
));
// A zero-fill during the compose step gives the "all-Neon-like" OOB-is-0 semantics
let compose = idxs.simd_lt(high).select(hi_shuf, Simd::splat(0));
let lolo = avx2_cross_shuffle::<0x00>(bytes.into(), bytes.into());
let lo_shuf = Simd::from(avx2_half_pshufb(lolo, idxs.into()));
// Repeat, then pick indices < 16, overwriting indices 0-15 from previous compose step
let compose = idxs.simd_lt(mid).select(lo_shuf, compose);
compose
```

That's pretty clever. The core insight is this: 

CPUs have a bunch of different hardware curcuits, and the circuit that handles shuffles is entirely separate from a circuit that handles arithmetic, so you can run them both at the same time. This is known as "[instruction-level parallelism](https://en.wikipedia.org/wiki/Instruction-level_parallelism)".

The above code is carefully tuned to spread the work across different circuits in such a way that it doesn't pile too much work on a single circuit (aka "port") and take advantage of this parallelism to speed things up.

You can measure the effectiveness of this by viewing the resulting assembly with [cargo-show-asm](https://crates.io/crates/cargo-show-asm) and feeding the result to [llvm-mca](https://llvm.org/docs/CommandGuide/llvm-mca.html) (Machine Code Analyzer), which models the execution of the given assembly on a specific CPU model accounting for all of these effects.

According to llvm-mca, spreading operations across different kinds of ports is important on the early Intel CPUs with AVX2, namely Haswell and Broadwell. On the other end of the spectrum, recent Intel Tiger Lake and all AMD Zen generations are so stuffed with all kinds of circuits that they favor doing less work overall, and you don't have to worry about overwhelming one circuit anymore. Intel Skylake is indifferent, dealing with all formulations in an equally mediocre fashion.

It's easy to get distracted arguing over which platform should be prioritized and optimized for. On one hand, recent CPUs are much more common these days, so optimizing for them benefits more people. On the other, the older CPUs are underpowered by today's standards and need all the help they can get.

We can sidestep that entirely by using a clever algorithm that eliminates the compare-select steps, which improves performance on recent CPUs without regressing it on antiquated ones:

```rust
use x86::_mm256_permute2x128_si256 as avx2_cross_shuffle;
use x86::_mm256_shuffle_epi8 as avx2_half_pshufb;

let lolo = avx2_cross_shuffle::<0x00>(bytes.into(), bytes.into());
let hihi = avx2_cross_shuffle::<0x11>(bytes.into(), bytes.into());

// Adding 0x60 preserves the low nibble and bit 4 for valid
// indices 0..=31. Larger indices get their high bit set, so
// VPSHUFB supplies the required out-of-bounds zeroing.
let control = x86::_mm256_adds_epu8(idxs.into(), x86::_mm256_set1_epi8(0x60));

// Move index bit 4 into each byte's sign bit for VPBLENDVB.
let select_high = x86::_mm256_slli_epi16::<3>(control);
let from_low = avx2_half_pshufb(lolo, control);
let from_high = avx2_half_pshufb(hihi, control);
x86::_mm256_blendv_epi8(from_low, from_high, select_high).into()
```

llvm-mca shows no change on Haswell and Broadwell, but Zen 1, Zen 2 and Tiger Lake show a 23% improvement, and a 7-20% improvement on Zen 3. Skylake is unchanged for the 256-bit shuffle, but the decomposition of a 512-bit shuffle into four 256-bit ones improves by 20%. 

*I'm glossing over the [throughput vs latency](https://www.geeksforgeeks.org/computer-organization-architecture/computer-organization-and-architecture-pipelining-set-1-execution-stages-and-throughput/) distinction here, since this formulation improves both.*

This change is currently [under review](https://github.com/rust-lang/portable-simd/pull/542) in `std::simd`, and `fearless_simd` will ship with this formulation from the start.

## Was it worth it?

Performance of a single SIMD operation in isolation does not necessarily translate to improvements in algorithms using it.

For example, AVX2 only has 16 SIMD registers to hold data the CPU can directly operate on. If your operation runs faster but uses a lot more registers, the entire algorithm it's in may run out of registers and have to spill data onto the stack, which adds costly load/store instructions, which negates the gains or even slows down the entire algorithm!

We need to measure performance of some plausible code that uses 512-bit swizzles. 512 bits is 64 bytes, so [base64](https://en.wikipedia.org/wiki/Base64) encoding/decoding is a natural fit.

I'm not aware of any shuffle-based Rust implementations of base64 that I could simply reuse. But it's easy enough to have an LLM conjure a [prototype](https://github.com/Shnatsel/intrepid-base64), for educational purposes only. Here's how it performs with different shuffle implementations:

```
scalar fallback:   1.6 GiB/s encode, 0.9 GiB/s decode
AVX2 double-width: 8.2 GiB/s encode, 5.5 GiB/s decode
AVX2 dw optimized: 9.4 GiB/s encode, 5.9 GiB/s decode
```

That's a ~6x improvement just from the work described in this article!

Let's see how it stacks up against [Sunny Young's base64 using std::simd](https://github.com/mcy/vb64/), which uses a lot of clever tricks and you should totally go read [their article about it](https://mcyoung.xyz/2023/11/27/simd-base64/):

```
AVX2 vb64: 5.9 GiB/s encode, 5.7 GiB/s decode
```

Brute-forcing the work with emulated swizzles and landing in the same ballpark as an implementation that carefully avoids them using [a clever perfect hash](https://mcyoung.xyz/2023/11/27/simd-base64/#simd-hash-table) is not too shabby! And ours have the added benefit of swapping out the base64 alphabet at will, while `vb64`'s perfect hash is tied to the standard base64 alphabet.

A [production implementation without SIMD](https://crates.io/crates/base64) runs at 2.7 GiB/s encode and 2.3 GiB/s decode, so SIMD is well worth it. Our implementation on an AVX-512 CPU runs at ~13 GiB/s, while vb64 doesn't benefit from AVX-512 nearly as much. A [state-of-the art](https://arxiv.org/abs/1910.05109) AVX-512 implementation [translated to safe Rust](https://github.com/Shnatsel/intrepid-base64/blob/main/src/avx512.rs) runs at 35 GiB/s encode and 20 GiB/s decode, so our prototype is not optimal, but still lets us see improvements to individual operations benefit a larger algorithm.

I've run these measurements on my Zen 4 workstation on `fearless_simd`, because the contributing guide for `std::simd` doesn't say how to benchmark your changes.

## Is fearless_simd better?

For this one specific operation, `swizzle_dyn`, yes. The implementations are now identical in both `std::simd` and `fearless_simd`, but the latter doesn't require `-Z build-std` and can multiversion this function just fine.

But when `std::simd` is playing to its strengths, it is really hard to beat or even match.

For example, [`std::simd::swizzle`](https://doc.rust-lang.org/stable/std/simd/trait.Swizzle.html#method.swizzle) that accepts indices known at compile time translates directly into an LLVM "shuffle vector" operation, and LLVM selects the best formulation for your CPU out of a long list of special cases.

Replicating this on stable Rust would require a lot of work, and either a lot of const evaluation or a procedural macro, both of which would negatively impact build time. To the best of my knowledge, nobody has even attempted this yet.

That said, LLVM is not perfect. For example, I've found that it destroys performance of certain AVX-512 shuffles by turning them into a lot of SSE operations instead of just keeping it as a native AVX-512 shuffle. I've [reported this as an issue in Rust](https://github.com/rust-lang/rust/issues/156891), then a Rust standard library maintainer [reported it to LLVM](https://github.com/llvm/llvm-project/issues/199445) and [attempted to fix it](github.com/llvm/llvm-project/pull/199454), and that led to a [whole](https://github.com/llvm/llvm-project/pull/201618) [bunch](https://github.com/llvm/llvm-project/pull/201781) of [improvements](https://github.com/llvm/llvm-project/pull/201831) made by LLVM developers. This work will ship in LLVM 23. 

I've already uncovered [another LLVM inefficiency](https://github.com/rust-lang/rust/issues/156946), but [the PR to fix it](https://github.com/llvm/llvm-project/pull/199633) just sends LLVM into an infinite loop. So here's a fun debugging puzzle if you're looking for one!

LLVM issues affect `std::simd`, all other SIMD crates, and even other languages like C and C++, so fixing them is always a big win.

## Takeaways

So here's what I've learned:

- `std::simd::swizzle_dyn` is even less useful than it appears, since multiversioning doesn't work on it.
- `std::simd` on nightly is outdated compared to the actual development state.
- The contributing documentation for `std::simd` is poor:
   - It's easy to open PRs for an outdated version in the wrong repository.
   - There's no documented way to disassemble or benchmark your changes.
- LLVM can mangle perfectly optimal SIMD code, even if you use intrinsics that are supposed to lower to a specific instruction. Not just in Rust, in C and C++ too!

And, well, `std::simd::swizzle_dyn` will be up to 6x faster once this work gets synced into the standard library. You're welcome!