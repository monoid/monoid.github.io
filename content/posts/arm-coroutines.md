---
title: "Rust Coroutines on AArch64 (ARM64)"
date: 2025-05-20T19:08:55+02:00
draft: true
---

<!-- Problem: book -->
The amazing book [Asynchronous Programming in
Rust](https://www.packtpub.com/en-us/product/asynchronous-programming-in-rust-9781805128137)
by Carl Fredrik Samson has a chapter on implementating stackful coroutines in Rust for
AMD64 architecture, both Linux and MacOS.  However, for sake of simplicity no Aarch64 implementation was
provided.  I tried to port it to Aarch64, and here's what I've got. 
<!-- TODO the source github repo link -->


The original code is available at https://github.com/PacktPublishing/Asynchronous-Programming-in-Rust\, see `ch05/c-fibers` directory.

<!-- ARM (AArch64) special feature: LR link register -->
## 1. AArch64 ABI
The AArch64 ABI is described in the "Procedure Call Standard for the Arm 64-bit Architecture" document found here:
https://github.com/ARM-software/abi-aa/releases\.

The chapter "6. The Base Procedure Call Standard" declares:

1. The registers `r19`..`r28` are *callee-saved* registers.
2. Subroutine call doesn't push return address to a stack, but uses `r30` register aka `LR`; its value is *caller-saved*.
3. `SP` mod 16 == 0.  The stack is always 16-byte aligned.
4. `r16` and `r17` are scratch registers. `r18` is "Platform Register, if needed; otherwise a temporary".
5. `r30` is a frame pointer register.

It worth checking MacOS-specific ABI details:
https://developer.apple.com/documentation/xcode/writing-arm64-code-for-apple-platforms\.  However there is nothing specific for our implementation, except confirming that `r18` should be ignored.

Please note that we ignore here floating point and vector register for simplicity, as the original implementation does.

### 1.1. Bits of Aarch64 assembler

Assembler instructions set used in the coroutine engine is quite small.

{{< mytable >}}
AMD64 syntax    | Aarch64 syntax    | Comment
--------------- | ----------------- | -------
`mov Target, Source`   | `mov Target, Source`    | Moving value from register to register
`mov Target, [Source + offset]`   | `ldr Target, [Source, offset]`    | Loading register from memory with offset
`mov [Target + offset], Source`   | `str Source, [Target, offset]`    | Storing register to memory with offset
`call Target`   | `blr Target`      | Subroutine call (but semantics is different)
`ret`           | `ret`             | Return from a subroutine (but semantics is different)
{{< /mytable >}}

The instruction for moving , loading and storing values are more or less same.
Subroutine calls, however, are very different in Aarch64.

### 1.2. Link register LR

When an x64 processor executes a `call` instruction, it pushes a return address
(i.e. the address of the next instruction) to the stack.  Unlike x64, Aarch64
has a dedicated register for that, `LR`.

A `ret` instruction also uses `LR` for return instead of popping an address
from the stack.  We might use a `br LR` instruction (branch to register `LR`),
but `ret` gives a *hint* to processor that this is a return from a `blr`
subroutine call.

But how can we call a subroutine from a subroutine?  The `blr` will overwrite
the `LR` value.  Well, the `LR` register is a caller-saved, so the parent
subroutine should store `LR` before calling any subroutine.  Usually, stack is
used for that.  If we look closely at it, a `x64` program with N nested calls
will push a return address to stack N times, and `Aarch64` program will store a
return address N-1 times.  Not a big deal.

However, while we used *push and `ret`* trick to switch coroutines, we cannot
use it at Aarch64.  We have to restore the `LR` register instead which requires
some more assembler code.

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
