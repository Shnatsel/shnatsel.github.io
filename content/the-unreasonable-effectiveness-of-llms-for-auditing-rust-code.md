+++
title = "The unreasonable effectiveness of LLMs for auditing Rust code"
date = 2026-06-20
description = "Using LLMs and Miri to find memory safety bugs in Rust crates."

[taxonomies]
tags = ["Rust", "Security", "LLMs", "Auditing"]

[extra]
author = 'Sergey "Shnatsel" Davidoff'
+++

As a lead of the [Rust Secure Code Working Group](https://rust-lang.org/governance/teams/#team-wg-secure-code), I got free access to GPT-5.5 via the [Codex for Open Source](https://openai.com/form/codex-for-oss/). Since then I’ve found and reported dozens of issues of varying severity in widely used Rust crates.

Separately, the [Rust Foundation security initiative](https://rustfoundation.org/security-initiative/) got access to [Mythos](https://www.anthropic.com/claude/mythos) via [Project Glasswing](https://www.anthropic.com/glasswing), and their report should also be coming soon. I’ve coordinated with them so that our audit targets would not overlap.

While I haven’t found any truly devastating vulnerabilities, I am very impressed with GPT-5.5 for auditing Rust source code, and I’ll absolutely be adding it to my toolkit alongside fuzzers.

_**Note:**_ _All opinions expressed in this article are my own, not that of any organizations I am a part of._

Methodology
-----------

Since maintainers could already be dealing with a large amount of vulnerability reports, it is _imperative_ that I do not submit any invalid vulnerability reports and waste maintainers’ already limited time.

So I’ve decided to look for unambiguously problematic class of vulnerability that’s easy to verify: memory safety bugs.

### Wait, isn’t Rust memory-safe?

Yes, with an asterisk.

Most code you’d write in Rust is memory-safe, but at some point you have to talk to the operating system or a C library or implement things like intrusive data structures, all of which involves raw pointers.

Most languages implement these parts in C (e.g. CPython), provide unsafe interoperability with C, and have you write C for your own unsafe code, while Rust has its own unsafe subset where you can muck about with raw pointers.

This puts Rust’s memory safety properties on par with Python’s, ahead of Go [which violates safety on data races](https://www.ralfj.de/blog/2025/07/24/memory-safety.html), and behind browser-sandboxed JavaScript (but you can match that by compiling Rust to WebAssembly).

As for the amount of safe vs unsafe code in the wild, [my own scan from 2020](https://www.reddit.com/r/rust/comments/g0wu9b/percentage_of_unsafe_code_per_crate_for/) showed that 95% of the code on [crates.io](http://crates.io) is memory safe. The authors of the 2020 paper “[How do programmers use unsafe Rust?](https://pm.inf.ethz.ch/publications/AstrauskasMathejaMuellerPoliSummers20.pdf)” independently arrived to the 95% number, although they didn’t put it into the final paper because they weren’t confident in their methodology for it. My own scan is also rather crude, but two completely different measurements arriving to the same number is encouraging.

In practice memory safety vulnerability rate reduction compared to C++ is [about 1000x](https://blog.google/security/rust-in-android-move-fast-fix-things/), which is more than you’d expect based on the above figures.

### Preventing false positives

Rust has a tool called [miri](https://github.com/rust-lang/miri) that runs Rust code in an interpreter and tells you precisely whether it committed any crimes against the language rules or not. Safe Rust cannot violate them by construction, but unsafe Rust can, and a validator that immediate tells you whether you messed up or not instead of having to parse dozens of pages of dense prose is indispensable.

It also **completely eliminates false positives** from LLM vulnerability findings.

If the LLM can construct a unit test that causes miri to fail, I can report that to the maintainers and be certain that it’s a bug. I don’t ever have to argue if it’s a real issue or not, either — the proof is right there. And if miri says the execution is completely fine, then the LLM false positive gets discarded before anyone even sees it.

To the best of my knowledge, **no other language** has a practical tool with this level of precision. [Sanitizers](https://clang.llvm.org/docs/AddressSanitizer.html) are very nice, but can’t catch everything, so verifying against them does not prove absence of issues.

Sadly miri is not without limitations — execution with extra checks is slow, calling into C is not supported, and syscall support is limited. When miri is not applicable, you can fall back on the [sanitizers](https://doc.rust-lang.org/beta/unstable-book/compiler-flags/sanitizer.html) and get _some_ filtering.

I also had to switch miri to the newer [Tree Borrows](https://perso.crans.org/vanille/treebor/) aliasing model (as opposed to the older [Stacked Borrows](https://plv.mpi-sws.org/rustbelt/stacked-borrows/)) to avoid false positives, but fortunately that’s just one flag, -Zmiri-tree-borrows.

### Harness

My setup was very basic: just Codex and a [prompt](https://gist.github.com/Shnatsel/e83219d7d6b73255373c2818ee438cda), with GPT-5.5 set to xhigh reasoning effort. It is important for the model to be able to write and run unit tests to try and trigger the issue under miri, so I consider something like Codex essential.

It would be interesting to try a more elaborate harness like [metis](https://github.com/arm/metis), but even this basic setup was enough to discover interesting bugs.

Findings
--------

The most serious issue I’ve found is an [out-of-bounds write in](https://github.com/tirr-c/jxl-oxide/security/advisories/GHSA-5pmv-rx8r-wmv5) [jxl-grid](https://github.com/tirr-c/jxl-oxide/security/advisories/GHSA-5pmv-rx8r-wmv5) [crate](https://github.com/tirr-c/jxl-oxide/security/advisories/GHSA-5pmv-rx8r-wmv5). It is a part of [jxl-oxide](https://github.com/tirr-c/jxl-oxide), a JPEG XL decoder in Rust (not to be confused with [jxl-rs](https://github.com/libjxl/jxl-rs), which Firefox and Chromium are adopting for JPEG XL decoding; that one came up clean in my audit).

I’ve already fuzzed this crate earlier, but the fuzzer didn’t catch this issue because it only happens on 32-bit platforms and requires very large image dimensions. I wasn’t running the fuzzer on 32-bit, and the fuzzer is limited to small image dimensions to avoid exhausting my computer’s RAM, so it never had a chance to trigger this condition at runtime.

The initial demonstrator showed this as a jxl-grid issue, but it was not clear if it’s a theoretical problem or if it can be triggered by decoding a crafted image. GPT-5.5 [helped analyze that too](https://gist.github.com/Shnatsel/2c4e4f75e5892988d1315aa7ede4e575), and it turned out to be reachable. This was very valuable information to correctly prioritize the bug.

This still isn’t that big a deal in practice because old 32-bit devices and very recent and computationally intensive image formats rarely meet, but it does showcase the capability of the tool quite well.

Here are some more samples of bugs GPT-5.5 has discovered, showing the breadth of the kinds issues it has found:

*   [Use-after-free](https://github.com/Amanieu/intrusive-rs/pull/104), [data races](https://github.com/Amanieu/intrusive-rs/pull/105) and [panic safety issues](https://github.com/Amanieu/intrusive-rs/pull/106) in intrusive-collections
    
*   [Out-of-bounds reads](https://github.com/rkyv/rkyv/issues/670) on deserializing crafted archives in rkyv
    
*   [Exposing uninitialized memory in serialized data](https://github.com/anza-xyz/wincode/issues/306) in wincode
    
*   [Incorrect](https://github.com/unicode-org/icu4x/pull/8029) [Send](https://github.com/unicode-org/icu4x/pull/8029)[/](https://github.com/unicode-org/icu4x/pull/8029)[Sync](https://github.com/unicode-org/icu4x/pull/8029) [impls](https://github.com/unicode-org/icu4x/pull/8029) in yoke
    
*   [Construction of invalid enum values](https://github.com/unicode-org/icu4x/pull/7940) in zerovec
    
*   [Data races for types with interior mutability](https://github.com/djc/hashlink/issues/42) in hashlink
    
*   [Soundness issue in Gecko FFI](https://github.com/mozilla/thin-vec/issues/86) in thin\_vec (crossing languages!)
    
*   [Out-of-bounds reads](https://github.com/jaredforth/webp/pull/51) in webp and [another one](https://github.com/imazen/webpx/commit/373015705ec84460ddc8722550805520478a2d57) inwebpx (C library wrappers)
    
*   [Turning a](https://github.com/zkat/miette/issues/469) [&](https://github.com/zkat/miette/issues/469) [into a](https://github.com/zkat/miette/issues/469) [&mut](https://github.com/zkat/miette/issues/469) in miette
    
*   [Multiple](https://github.com/ruffle-rs/nihav-vp6/issues/2) [&mut](https://github.com/ruffle-rs/nihav-vp6/issues/2) [pointing to the same memory](https://github.com/ruffle-rs/nihav-vp6/issues/2) in nihav-core
    
*   [Aliasing violation](https://github.com/andylokandy/arraydeque/issues/34) in arraydeque
    
*   [Incorrect alignment handling](https://github.com/rust-av/v_frame/pull/74) in v\_frame
    
*   An interesting type confusion issue that’s not yet public
    

This demonstrates not just understanding of generic issues like out-of-bounds accesses, but also the ability to reason about Rust-specific concepts such as panic safety, aliasing, and the Send/Sync traits that enforce thread safety.

I also got some results out of Claude before I got GPT-5.5 access:

*   [Use-after-free](https://github.com/servo/rust-smallvec/pull/407) in one function on a zero-capacity SmallVec (Opus 4.6)
    
*   [Multiple out-of-bounds reads](https://rustsec.org/packages/imageproc.html) in imageproc (Opus 4.7)
    

I haven’t used Claude enough to be able to compare the models. I also cannot compare GPT-5.5 to Mythos, as interesting as that would be, because I deliberately picked different targets to avoid duplicate vulnerability reports putting extra load on maintainers.

Fixes
-----

Once the issue is identified and explained, the model usually can also fix it autonomously. I avoid “fix the bugs you found” style prompts and instead discuss possible solutions with the model first, then have the model implement one of them.

Submitting a possible fix alongside the vulnerability report puts less pressure on the maintainers. If you look up my pull requests, the first commit usually adds proof-of-concept tests that cause miri to complain, and the subsequent commit fixes the issues and turns the proof-of-concept snippets into regression tests.

GPT-5.5 has also assisted me in locating the version where the bug was introduced, which is essential for security advisories.

Reflections
-----------

In my experience Rust code is a lot easier to audit than C code. In C, if I look at a line like data\[a + b\], I have to trace through the entire codebase and find all the possible values it can be set to just to validate this one line and make sure it doesn't have out-of-bounds accesses.

Rust, even unsafe Rust, still relies on local reasoning: if I see unsafe { data.get\_unchecked(a + b) }, then the function it's in must either validate a and b to make sure their addition is in-bounds, or be itself unsafe to call. Either way, there is a [clear point where verification must happen](https://kobzol.github.io/rust/2026/06/15/how-memory-safety-cves-differ-between-rust-and-c-cpp.html) - and if it's not there, it's a bug.

I don’t have to chase the data flow through the entire JPEG XL decoder by hand, and neither does an LLM. This reduces the complexity of auditing the code from combinatorial (all possible combinations of call trees) to linear (each function in isolation).

In that light, it’s not terribly surprising that LLMs are so good at auditing Rust code. And it’s also not terribly surprising that I haven’t found any devastating vulnerabilities after all.

Limitations
-----------

Just like fuzzers before them, LLMs surface numerous bugs that weren’t economical to discover previously. But at the end of the day, no heuristic tool can prove the absence of vulnerabilities.

For example, a GPT-5.5 alone didn’t discover [several bugs](https://github.com/mariofeter/secureloop-findings-public/tree/master/findings/rkyv) that a combination of a simpler LLM with a fuzzer did.

But we don’t have to rely on heuristics. Rust without unsafe does guarantee the absence of memory safety bugs.

So I’ll [keep](https://shnatsel.medium.com/how-to-avoid-bounds-checks-in-rust-without-unsafe-f65e618b4c1e) [shrinking](https://shnatsel.medium.com/safe-simd-in-rust-even-on-the-inside-c6f1ff381828) the unsafe surface where I can, and I'm glad to have these tools for when I can't.

*This article was originally published [on my Medium](https://shnatsel.medium.com/the-unreasonable-effectiveness-of-llms-for-auditing-rust-code-d4df8bf0afd3)*