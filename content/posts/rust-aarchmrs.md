---
title: "AARCHMRS dataset for AArch64 instruction synthesis in Rust"
date: 2025-07-29T16:12:02+02:00
draft: false
tags: ["Arm", "AARCHMRS", "Rust"]
---
Arm Limited, the company behind the ARM architecture, publishes [AARCHMRS
dataset](https://developer.arm.com/Architectures/A-Profile%20Architecture#Downloads)
-- "Arm Architecture Machine Readable Specification", a detailed description of
an ARM processor as a set of JSON files.

Today I published a set of crates:

+ [`aarchmrs-parser`](https://crates.io/crates/aarchmrs-parser): a set of
  definitions that allows parsing the AARCHMRS dataset, but only instructions
  description file.
+ [`aarchmrs-instructions`](https://crates.io/crates/aarchmrs-instructions): a
  set of functions derived from the AARCHMRS that allow you to build
  instruction codes.
+ [`aarchmrs-types`](https://crates.io/crates/aarchmrs-types): a helper crate
  with useful types for `aarchmrs-instructions`, which now contains only generated code.

The license is the same as the original dataset: BSD-3-Clause.

The sources and generation tooling is available at https://github.com/monoid/harm/.

# Bits of history
When I experimented with this dataset, my initial version simply generated
functions for instruction synthesis[^synthesis].

Then I thought about a more flexible approach: what if, instead of a function, we
used a struct that holds the arguments, and we can change these arguments and then
finally build it?

My third approach was using various bit-field crates (every ARM instruction is
32-bit value, and using bit-fields seems to be a natural approach).
Unfortunately, it proved to be extremly slow: bit-field structs are implemented
with a proc-macro, and for each definition there is subprocess invocation that
handles the input code and produces output. It also generated huge amounts of
`target` data.

After some experiments I decided to switch back to function definitions to reduce
compilation time as much as possible.

But what impacted the compilation time most is `build.rs` file.  My original
idea was to generated the code with `build.rs`.  However, there is no point
generating it on every compilation: it would produce the same output.  So,
generation was started only when a specific environment variable was manually
set, but lot of dependencies for an HTTP client (because the `build.rs` would
automatically download the AARCHMRS file if it is not available or broken) would
compile anyway, even if it was not needed.  This is a huge dependency, so
eventually I removed the `build.rs` and created a tool (`aarchmrs-generate` in
the GitHub repository above) that generates code only when it is really needed
(when generator changes or when new AARCHMRS is released).

So, it takes now just a few seconds to compile `aarchmrs-instructions` on my good
old Macbook Air M1.  I'm quite satisfied with the result.

[^synthesis]: while "synthesis" sounds really smart, here it simply means
"computing something from set of inputs".
