---
title: "Rust Coroutines on ARM (AArch64)"
date: 2024-08-31T19:08:55+02:00
draft: true
---

<!-- Problem: book -->
The amazing book [Asynchronous Programming in Rust](https://www.packtpub.com/en-us/product/asynchronous-programming-in-rust-9781805128137) by Carl Fredrik Samson has an implementation of stackful coroutines in Rust for AMD64 architecture, both Linux and MacOS.  However, no ARM64 implementation was provided to make the implementation simpler.  I tried to port it to ARM64, and here's what I've got. TODO the source github repo link

The original code is available at https://github.com/PacktPublishing/Asynchronous-Programming-in-Rust\, see `ch05/c-fibers` directory.

<!-- ARM (AArch64) special feature: LR link register -->
## 1. AArch64 ABI
The AArch64 ABI is described in the "Procedure Call Standard for the Arm 64-bit Architecture" document found here:
https://github.com/ARM-software/abi-aa/releases\.

The chapter "6. The Base Procedure Call Standard" declares following:

1. The registers `r19`..`r28` are *callee-saved* registers.
2. Subroutine call doesn't push return address to a stack, but uses `r30` register aka `LR`; its value is *caller-saved*.
3. `SP` mod 16 == 0.  The stack is always 16-byte aligned.

It worth checking MacOS-specific ABI details:
https://developer.apple.com/documentation/xcode/writing-arm64-code-for-apple-platforms\.  However there is nothing specific for our implementation, except confirming that `r18` should be ignored.

Please note that we ignore here floating point and vector register for simplicity, as the original implementation does.

### 1.1. Bits of ARM64 assembler

### 1.2. Link register LR

{{< mytable >}}
AMD64 syntax | ARM64 syntax | Comment
------------ | ------------ | -------
`mov Target, Source`   | `mov Target, Source`    | Moving value from register to register
`mov Target, [Source + offset]`   | `ldr Target, [Source, offset]`    | Loading register from memory with offset
`mov [Target + offset], Source`   | `str Source, [Target, offset]`    | Storing register to memory with offset
{{< /mytable >}}

## Setting up restoring context
<!-- Using trampoline function -->

<!-- write example with PUSH and POP
     problem: there is no PUSH and POP in Rust AArch64 assembler
-->
<!-- Final result  -->

<!-- N. Misc -->
<!-- N.1. Red zone for state saving on MacOS -->
<!--  -->

<!-- Local Variables: -->
<!-- spell-fu-buffer-session-localwords: ("coroutine" "coroutines" "MacOS" "stackful") -->
<!-- End: -->
