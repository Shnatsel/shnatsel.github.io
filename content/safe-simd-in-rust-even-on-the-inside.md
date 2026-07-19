+++
title = "Safe SIMD in Rust, even on the inside"
date = 2026-06-19
description = "Rust's SIMD abstractions were not as safe as I'd like. Until now."

[taxonomies]
tags = ["Rust", "SIMD", "Unsafe Rust"]

[extra]
author = 'Sergey "Shnatsel" Davidoff'
+++

_Rust’s_ [_SIMD abstractions_](https://shnatsel.medium.com/the-state-of-simd-in-rust-in-2025-32c263e5f53d) _were not as safe as I’d like. Until now._

It’s no secret that raw SIMD intrinsics are unpleasant to use.

You want to write `a + b`, not this monstrosity:

```rust
unsafe {
    #[cfg(all(any(target_arch = "x86", target_arch = "x86_64"), target_feature = "avx2"))]
    _mm256_add_ps(a, b)
    #[cfg(all(any(target_arch = "x86", target_arch = "x86_64"), target_feature = "sse", not(target_feature = "avx2")))]
    _mm_add_ps(a, b)
    #[cfg(all(target_arch = "aarch64", target_feature = "neon"))]
    vaddq_f32(a, b)
}
```

Look at it. It’s hideous. And the whole thing is wrapped in `unsafe`!

And that’s a simplified example. It still doesn’t handle:

* Other common platforms: AVX-512, 32-bit ARM, WebAssembly
* Platforms without SIMD or obscure platforms like RISC-V
* Actually loading data like `&[f32]` into a form that each intrinsic accepts
* Selecting the best implementation for the CPU it’s running on

Luckily, Rust provides [many SIMD abstractions](https://shnatsel.medium.com/the-state-of-simd-in-rust-in-2025-32c263e5f53d) that handle all of that for you and let you simply write `a + b`.

There is just one wrinkle. Inside, they’re still full of `unsafe`. It wasn’t gone, just hidden. Vast quantities of it lurking just beneath the surface, getting [screwed](https://github.com/servo/pathfinder/issues/588) [up](https://github.com/wingertge/macerator/issues/31) [occasionally](https://github.com/sarah-quinones/pulp/issues/28).

Or rather, they were. Until now.

## Why do we even need ‘unsafe’?

For the longest time you couldn’t get around wrapping the call to each intrinsic function such as `_mm256_add_ps` into `unsafe` because it is illegal to call one when it’s not available on the CPU you’re running on.

So you _had_ to have some mechanism for tracking which instructions are needed for each intrinsic, and which instructions you have access to, and cross-referencing them to decide if it’s safe to call a given function.

It was either tedious if done by hand or complex if done by a code generator, always error-prone, and required `unsafe` around every intrinsic.

This changed in Rust 1.87 when the compiler started tracking the required instruction sets itself, so you could write this:

```rust
#[target_feature(enable = "avx2")]
fn add_avx2(a: __m256, b: __m256) -> __m256 {
    _mm256_add_ps(a, b) // this is an avx2 intrinsic
}
```

Look ma, no `unsafe`!

…yet.

You still cannot write `a + b` with this. The best you can do is this:

```rust
unsafe { add_avx2(a, b) }
```

This only shifts the `unsafe` up a layer. You can call intrinsics inside functions annotated with the correct `#[target_feature]` now, but there still has to be `unsafe` somewhere in the chain.

The other problem is more fundamental. You cannot put `#[target_feature]` on the implementation of `+` for your type, because `+` must be available always. So no `a + b` for us using this mechanism.

## Lemma: CPU feature tokens

To understand how the final solution works, you first need to understand how CPU feature detection works.

Normally, checking for a CPU feature like AVX2 is done at runtime using `is_x86_feature_detected!("avx2")`. But we definitely don’t want to run this check every single time we add two numbers together — that would completely tank performance. We want to check it _once_, and then prove to the compiler that it’s safe to use AVX2 instructions from that point on.

Instead we can encode this proof into the type system using an _unforgeable token_: a zero-sized type with a private inner field. The _only_ way to obtain this token is to call a function that performs the CPU feature check. If the check passes, the function hands you the token:

```rust
pub struct Avx2(());

fn detect_avx2() -> Option<Avx2> {
    if is_x86_feature_detected!("avx2") {
        Some(Avx2(()))
    } else {
        None
    }
}
```

And because it’s a zero-sized type, passing this token around has no runtime overhead. It exists purely as a compile-time proof.

The upshot is that as long as you have an instance of the `Avx2` struct, you can be sure that AVX2 instructions are available on the system.

## The key insight

The compiler doesn’t know it, but this function is safe to call:

```rust
#[target_feature(enable = "avx2")]
fn add_avx2(token: Avx2, a: __m256, b: __m256) -> __m256 {
    _mm256_add_ps(a, b)
}
```

You can _only_ call this function if you have an `Avx2` token, which you can only get if AVX2 instructions are available on the system.

If we can explain to the compiler that this is valid (using `unsafe`), we can write that `unsafe` only once and reuse it everywhere.

What we need is a macro which is safe to invoke:

```rust
with_avx2!(
    fn add_avx2(token: Avx2, a: __m256, b: __m256) -> __m256 {
        _mm256_add_ps(a, b)
    }
)
```

but expands into this behind the scenes:

```rust
fn add_avx2(token: Avx2, a: __m256, b: __m256) -> __m256 {
    // SAFETY: Avx2 is available according to the token,
    // and we verified that the inner function is not an `unsafe fn`
    unsafe { inner(token, a, b) }

    #[target_feature(enable = "avx2")]
    fn inner(token: Avx2, a: __m256, b: __m256) -> __m256 {
        _mm256_add_ps(a, b)
    }
}
```

Now if you use an intrinsic that isn’t in AVX2, **the compiler will reject it!**

We’ve just managed to provide a safe programming interface to SIMD intrinsics without any bespoke tracking of target features!

Even though there is one `unsafe` block still inside it, it’s encapsulated in a sound API, so you cannot misuse it to cause memory safety bugs. In that sense it is just like `println!`, safely abstracting unsafe code.

This way you only ever need to review and audit _this one macro,_ not hundreds upon hundreds of bespoke `unsafe` blocks. And the only things we could _possibly_ screw up in the implementation are:

1. Mapping the token to the wrong `#[target_feature]`
2. Allowing an `unsafe fn` to be called from a safe context

And both failure modes are quite easy to check for.

So now we can call `add_avx2(token, a, b)` without `unsafe`, but that still doesn’t get us to `a + b`. How do we solve _that?_

## Generics to the rescue

We cannot annotate the implementation of `a + b` with `#[target_feature]` because it must be safe to call from anywhere. And we cannot pass a token into the function because it accepts `a` and `b` but not `token`.

But even if we could do that, it would make for a pretty ugly API. We want `a + b` to always work and automatically use the best SIMD instructions without the user ever fussing with tokens.

We can use generics to solve both problems at once: by defining an `f32x8` type that’s generic over the available instruction sets, we can implement addition on it that both smuggles a token inside it _and_ creates a separate implementation for each SIMD instruction set!

This is what it looks like:

```rust
pub trait Level {}

#[derive(Clone, Copy)]
pub struct Avx2(());
impl Level for Avx2 {}

pub struct f32x8<L: Level> {
    // For simplicity we'll back this with an array in this example.
    // In production code we use native SIMD types for the level.
    data: [f32; 8],
    // The smuggled token!
    token: L,
}

/// implementation of `a + b` for Avx2
impl std::ops::Add for f32x8<Avx2> {
    type Output = Self;

    fn add(self, rhs: Self) -> Self::Output {
        // (type conversions abbreviated)
        // Use the Avx2 token to call our safe wrapper
        let result = add_avx2(self.token, self, rhs);
        Self {
            data: store_m256(result),
            token: self.token,
        }
    }
}
```

And then we can just as easily make it work for any other instruction set, or when SIMD is not available at all:

```rust
#[derive(Clone, Copy)]
pub struct NoSimd(());
impl Level for NoSimd {}

/// implementation of `a + b` when no SIMD is available
impl std::ops::Add for f32x8<NoSimd> {
    type Output = Self;

    fn add(self, rhs: Self) -> Self::Output {
        let result = std::array::from_fn(|i| self.data[i] + rhs.data[i]);
        Self {
            data: result,
            token: self.token,
        }
    }
}
```

We’ve just solved safety _and_ runtime instruction selection at once!

Add a convenience function that gives you the best `Level` available on the system, and you get pretty much the perfect API for SIMD!

## The ABI would like a word

There is, unfortunately, a fundamental problem to writing `a + b` and having it lower to SIMD instructions: **function call overhead.**

Calling a function is not free but pretty cheap — just a handful of CPU instructions. But a _handful_ of instructions is a lot more than _one_ instruction we’ve just used for implementing addition!

So if there’s a function call in the way, addition performance will plummet. And performance is the entire point of using SIMD in the first place!

The compiler is usually pretty good at erasing this overhead via [inlining](https://matklad.github.io/2021/07/09/inline-in-rust.html). It basically copy-pastes the implementation of a function you called into the function calling it, so there’s no more function and no more overhead.

But `#[target_feature]` annotations throw a wrench in the works. The compiler cannot inline a function that has a `#[target_feature]` annotation into one that doesn’t, because the required features are not available in it!

And guess what cannot have a `#[target_feature]` annotation? Yeah.

So how _do_ we make `a + b` work with SIMD?

## Inline it, inline it with fire!

We cannot put `#[target_feature]` on the function that implements `a + b`, but we _can_ put it on the function that calls `a + b`!

Then we can use [inlining](https://matklad.github.io/2021/07/09/inline-in-rust.html) to get the implementation of `a + b` copied into the function calling it, and it ends up in a `#[target_feature]` context.

So the call chain looks like this:

```rust
#[target_feature(enable = "avx2")]
fn do_stuff() {
    // TODO: some computation
    c = a + b;
    // TODO: some more computation
}

// which calls into...

#[inline(always)] // Function body will be copied into the caller
fn add(self, rhs: Self) -> Self::Output {
    // Use the Avx2 token to call our safe wrapper
    add_avx2(self.token, self, rhs);
    // return statement abbreviated
}

// which calls into...

#[inline] // Function body will be copied into the caller if feasible
#[target_feature(enable = "avx2")]
fn add_avx2(token: Avx2, a: __m256, b: __m256) -> __m256 {
    _mm256_add_ps(a, b)
}
```

This works.

You can add these annotations in the right places and abstract over the SIMD levels and you [don’t even need macros](https://docs.rs/fearless_simd/latest/fearless_simd/trait.Simd.html#tymethod.vectorize).

The problem is that whenever you call `a + b` on SIMD types, you have to do it from a function with either `#[inline(always)]` or `#[target_feature]` on it, otherwise the code still compiles but performance plummets.

Wanna see for yourself? Open [this example](https://rust.godbolt.org/z/xo1fEfqnT), remove `#[target_feature]` from it and watch the generated assembly turn into absolute horror show.

I’m not sure what can be done about this. The limitation seems quite fundamental for any approach that implements `a + b` with SIMD.

The [Struct Target Features RFC](https://shnatsel.medium.com/github.com/rust-lang/rfcs/pull/3525) solves this for `add_avx2(token, a, b)` and `add::<S: Simd>(token, a, b)` but I don’t see a path to `a + b` just yet.

## Where does that leave us?

Despite the inlining wrinkle inherent to all SIMD code, we’ve managed to provide a remarkably pleasant SIMD abstraction at a staggeringly low, never-before-seen level of unsafe code.

You can find the production version of these ideas in [`fearless_simd`](https://crates.io/crates/fearless_simd) v0.5, **available now** in a package registry near you! And [here’s a small example](https://github.com/linebender/fearless_simd/blob/d2411665f0726cd6c09bd6fd98af4af18e3c1778/fearless_simd/examples/sigmoid.rs) to see how it all fits together in production.

The macro used to implement it is also exposed publicly, so you can easily [mix and match](https://github.com/linebender/fearless_simd/blob/e76b820154dc09f41bca8302670424d8ca079219/fearless_simd/examples/srgb.rs) high-level operations like `a + b` and platform-specific intrinsics to take full advantage of the hardware.

There is more than one `unsafe` block in `fearless_simd` because it also provides the functionality of [`safe_unaligned_simd`](https://crates.io/crates/safe_unaligned_simd) crate, but that too is done at a [significantly lower](https://github.com/okaneco/safe_unaligned_simd/issues/51) amount of unsafe code than the original.

For me the barrier to using high-level SIMD abstractions was always the sheer amount of `unsafe` they brought. It was scary and hard to justify.

But now SIMD in Rust can be truly fearless.

## Acknowledgements

I am shocked that I am the first to put this into production (as far as I can tell), because I’m certainly not the first person to think of this.

CPU feature tokens are an old and common idea. The [`pulp`](https://crates.io/crates/pulp) crate has been using them for years, but they relied on handwritten unsafe wrappers around intrinsics, and [occasionally got them wrong](https://github.com/sarah-quinones/pulp/issues/28).

Generating multiple implementations using generics is also an old idea. It is part of the [original `fearless_simd` concept](https://linebender.org/blog/towards-fearless-simd/) from 8 years ago. The [`simdeez`](https://crates.io/crates/simdeez) crate, which predates it, also [seems to use something similar](https://docs.rs/simdeez/1.0.8/simdeez/).

The key insight of combining tokens with a single safe wrapper that delegates to `rustc` is not unique to me either. Just in the context of `fearless_simd` crate, Raph Levien has [experimented with it](https://github.com/linebender/fearless_simd/commit/81b6ab4c8dd6f25064f539c49818c45c2f686815), and [Daniel McNab](https://github.com/DJMcNab) created a [more elaborate implementation](https://github.com/linebender/fearless_simd/pull/108) than mine months ago.

Daniel’s approach allows fine-grained tracking of every individual CPU feature, as opposed to a handful of fixed CPU feature levels that `fearless_simd` uses. It is more expressive, but came at the cost of complexity, and his approach never got merged because no other maintainer stepped up to review it. I still hope it will be published as a standalone crate someday.

Thanks to Daniel and to [Laurenz Stampfl](https://github.com/LaurenzV) for reviewing all my PRs to `fearless_simd`, they were big and the quick reviews are really appreciated!

*This article was originally published [on my Medium](https://shnatsel.medium.com/safe-simd-in-rust-even-on-the-inside-c6f1ff381828).*