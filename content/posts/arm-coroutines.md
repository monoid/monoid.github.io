---
title: "Rust Coroutines on AArch64 (ARM64)"
date: 2025-05-20T19:08:55+02:00
draft: false
tags: ["ARM", "Rust", "coroutine"]
---

The amazing book [Asynchronous Programming in
Rust](https://www.packtpub.com/en-us/product/asynchronous-programming-in-rust-9781805128137)
by Carl Fredrik Samson has a chapter on implementing stackful coroutines
(fibers) in Rust for `x86_64` architecture, both Linux and MacOS. But unlike
the other chapters, no `AArch64` implementation was included for the sake of
simplicity. I&nbsp;tried to port it to `AArch64`, and this post describes my
approach.

This text is just an additional material for the book. I&nbsp;presume you are
familiar either with it or with the original code at
[`ch05/c-fibers`](https://github.com/PacktPublishing/Asynchronous-Programming-in-Rust/tree/main/ch05/c-fibers).

Please note that inline asm usage in Rust has changed since the book was
written. The examples in the article use modern version of Rust with
`#[unsafe(naked)]` and `naked_asm!` naked function syntax.

The source code is available at the pull request at:
https://github.com/PacktPublishing/Asynchronous-Programming-in-Rust/pull/34\.

{{< toc >}}

## 1. AArch64 ABI -- how AArch64 does calls
Unlike `x86_64`, ARM's register bank is fairly uniform: it has 31 generic
registers (some of them have usage-specific aliases, though). Instructions
can also refer to the 32nd register, though: depending on particular instruction[^variant], it is
either `SP` (stack pointer) or `ZR` (a virtual zero register).

[^variant]: Instruction variant, actually! "Add (extended register)" can use
    `SP` as a destination or first argument, but "Add (shifted register)"
    allows `ZR`.

The AArch64 ABI is described in the "Procedure Call Standard for the Arm 64-bit
Architecture" document found here:
https://github.com/ARM-software/abi-aa/releases\.

The chapter "6. The Base Procedure Call Standard" declares:

1. The registers `r19`..`r28` are *callee-saved* registers.
2. `r29` aka `FP` is a frame pointer register, also *callee-saved*.
3. Subroutine call doesn't push return address to a stack, but uses `r30` register aka `LR`; its value is *caller-saved*.
4. `SP` mod 16 == 0.  The stack is always 16-byte aligned.
5. `r16` and `r17` are linker's scratch  registers. `r18` is "Platform Register, if needed; otherwise a temporary".
6. The rest of registers are *caller-saved*.

It's worth checking MacOS-specific ABI details:
https://developer.apple.com/documentation/xcode/writing-arm64-code-for-apple-platforms\.
However, there is nothing specific for our implementation, except confirming that
`r18` should be ignored.

The Linux ABI simply refers to the generic AArch64 ABI.

Please note that we ignore here floating point and vector register for
simplicity, just as the original implementation does.

### 1.1. Bits of AArch64 assembler

The instruction set used in the fiber engine is quite small.

{{< mytable >}}
`x86_64` syntax         | `AArch64` syntax     | Comment
----------------------- | --------------------- | -------
`mov Target, Source`    | `mov Target, Source`  | Moving value from register to register
`mov Target, [Source + offset]` | `ldr Target, [Source, offset]`    | Loading register from memory with offset
`mov [Target + offset], Source`   | `str Source, [Target, offset]`    | Storing register to memory with offset
`call Target`           | `blr Target`          | Subroutine call (but semantics is different)
`ret`                   | `ret`                 | Return from a subroutine (but semantics is different)
{{< /mytable >}}

The instruction for moving , loading and storing values are more or less same,
though naming is different. Subroutine calls and returns, however, are really
different in AArch64.

### 1.2. Link register LR

When an `x86_64` processor executes a **`call`** instruction, it pushes a return
address (i.e. the address of the next instruction) to the stack. Unlike
`x86_64,` AArch64 has a dedicated register for that, `LR`, and `blr` instruction
name means "branch with link register".

A **`ret`** instruction also uses[^ret_reg] `LR` as a return address instead of
popping it from the stack. It is almost equivalent to `br LR` instruction
(branch to register `LR`), but **`ret`** gives a *hint* to processor that this
is a return from a **`blr`** subroutine call, enabling better branch prediction.

[^ret_reg]: Unlike `x86_64`, `AArch64`'s `ret` takes an optional register
argument which defaults to `LR`.

But how can we call a nested subroutine? The nested **`blr`** will overwrite the
`LR` value. Well, the `LR` register is a caller-saved, so the parent subroutine
should store `LR` somewhere before calling any nested one. Usually, it is stored
on the `SP` stack. Using a method of careful inspection, we can immediately see
that a `x86_64` program with N nested calls will push a return address to stack
N times, and `AArch64` program will store a return address N-1 times[^offbyone].
Not a big deal.

[^offbyone]: An attentive reader may say that N nested calls require N-1 pushes
on `x86_64` and N-2 pushes on `AArch64`; it may true for normal threads like a
`main` function, but for our fiber we use **push-and-ret** trick to spawn
it, so there is an extra push at the very top of the stack.

However, while we used the **keep return address at the stack top and `ret`**
trick to spawn a fiber, we cannot use it at AArch64. While memory is still
the only place to keep `LR` between the switches, we have to save and restore
the `LR` register manually at spawn, and it requires some more assembler code.

## 2. Running a fiber

The `ThreadContext` is still a sequence of `u64` fields, one field for each
register to be saved and restored. Saving and restoring is also straightforward,
except `SP`, being a special register, cannot be stored or loaded from memory
directly and has to be stored to an intermediate register first (and restored
from an intermediate register).

```rust
#[derive(Debug, Default)]
#[repr(C)]
struct ThreadContext {
    r19: u64, // 00
    r20: u64, // 08
    r21: u64, // 10
    r22: u64, // 18
    r23: u64, // 20
    r24: u64, // 28
    r25: u64, // 30
    r26: u64, // 38
    r27: u64, // 40
    r28: u64, // 48
    fp: u64,  // 50
    lr: u64,  // 58
    sp: u64,  // 60
}
```

The `x86_64` version imitated nested calls: `f`, the fiber body, a technical
`skip` function and then the `guard` function.

Here, we don't need `skip` function to be run after `f`; we use the `trampoline`
function to be run before `f` instead.

### 2.1 Spawning a fiber
Launching a fiber is little more involved. We cannot just arrange stack to have
return addresses of functions at proper positions; we have to set `LR` to be
equal to `guard` and then jump to fiber function. The `ThreadContext` allows us
only to set `LR` register; let's save the values needed to the stack and set the
context's `LR` to a `trampoline` function that does the heavy lifting.

```rust
const F_TRAMPOLINE_OFFSET: usize = 0;
const GUARD_TRAMPOLINE_OFFSET: usize = F_TRAMPOLINE_OFFSET + std::mem::size_of::<u64>();

impl Runtime {
    // ...

    pub fn spawn(&mut self, f: fn()) {
        let available = self
            .threads
            .iter_mut()
            .find(|t| t.state == State::Available)
            .expect("no available thread.");

        let size = available.stack.len();

        // prepare stack for the `trampoline`
        let stack_top;
        unsafe {
            let s_end = available.stack.as_mut_ptr().add(size);
            let s_end_aligned = (s_end as usize & !15) as *mut u8;
            stack_top = s_end_aligned.offset(-32);
            std::ptr::write(stack_top.add(F_TRAMPOLINE_OFFSET).cast::<u64>(), f as u64);
            std::ptr::write(stack_top.add(GUARD_TRAMPOLINE_OFFSET).cast::<u64>(), guard as u64);
        }
        available.ctx.lr = trampoline as u64;
        available.ctx.fp = 0;
        available.ctx.sp = stack_top as u64;
        available.state = State::Ready;
    }
}

#[unsafe(naked)]
#[no_mangle]
unsafe extern "C" fn trampoline() {
    // The stack is prepared by the `Runtime::spawn`:
    // sp + 00      function_address
    // sp + 08      guard address
    // sp + 10 ..   unused
    naked_asm! {
        "ldr x1, [sp, {f_trampoline_offset}]",
        "ldr lr, [sp, {guard_trampoline_offset}]",
        "sub sp, sp, 0x10",  // current stack frame is not needed anymore
        "br x1",
        f_trampoline_offset = const F_TRAMPOLINE_OFFSET,
        guard_trampoline_offset = const GUARD_TRAMPOLINE_OFFSET,
    };
}
```

{{< callout title="Homework 2.1" >}}
Pass the values to the trampoline with `r0` and `r1` registers instead of the
stack. Does it make any performance difference? Does it make the code cleaner?
{{< /callout >}}

### 2.2 Switching contexts

As it was explained before, switching contexts is straightforward:
```rust
#[unsafe(naked)]
#[no_mangle]
unsafe extern "C" fn switch() {
    naked_asm! {
        // saving the old fiber.
        // TODO we might use `stp` instruction to store a pair of registers at once, but we don't.
        "str  x19, [x0, 0x00]",
        "str  x20, [x0, 0x08]",
        "str  x21, [x0, 0x10]",
        "str  x22, [x0, 0x18]",
        "str  x23, [x0, 0x20]",
        "str  x24, [x0, 0x28]",
        "str  x25, [x0, 0x30]",
        "str  x26, [x0, 0x38]",
        "str  x27, [x0, 0x40]",
        "str  x28, [x0, 0x48]",
        "str  fp, [x0, 0x50]",
        "str  lr, [x0, 0x58]",
        // sp cannot be stored/loaded directly -- use an intermediate register, one of the stored/loaded.
        "mov  x2, sp",
        "str  x2, [x0, 0x60]",

        // loading the new fiber
        // TODO we might use `ldp` instruction to load a pair of registers at once, but we don't.
        "ldr  x19, [x1, 0x00]",
        "ldr  x20, [x1, 0x08]",
        "ldr  x21, [x1, 0x10]",
        "ldr  x22, [x1, 0x18]",
        "ldr  x23, [x1, 0x20]",
        "ldr  x24, [x1, 0x28]",
        "ldr  x25, [x1, 0x30]",
        "ldr  x26, [x1, 0x38]",
        "ldr  x27, [x1, 0x40]",
        "ldr  x28, [x1, 0x48]",
        "ldr  fp, [x1, 0x50]",
        "ldr  lr, [x1, 0x58]",
        "ldr  x2, [x1, 0x60]",
        "mov  sp, x2",
        "ret",
    };
}
```

{{< callout title="Homework 2.2" >}}
Reimplement `switch` using `stp`/`ldp`. Use post-increment mode rather than
fixed offsets. If you don't know AArch64's post-increment syntax, ask your
mom.
{{< /callout >}}


Porting the example was not that difficult: the only obstacle was setting
the `LR` register instead of pushing addresses to the stack.  You may find it
instructive to port other examples of the Chapter 5!

## 3. Thoughts on a portable version

A trampoline simplifies creating a version that works on both architectures
(with proper `#[cfg]`s). On `x86_64`, a trampoline might look like this:

```rust
        // Set up in the `Runtime::spawn`:
        unsafe {
            stack_top = s_end_aligned.sub(std::mem::size_of::<u64>());
            std::ptr::write(stack_top.cast::<u64>(), trampoline as u64);
        }
        available.ctx.rax = f as u64;
        available.ctx.rbx = guard as u64;
        available.ctx.sp = stack_top;
        // ..

#[unsafe(naked)]
#[no_mangle]
unsafe extern "C" fn trampoline() {
    // The stack and registers are set by the `Runtime::spawn`:
    // 1. the stack is aligned by 16 bytes boundary
    // 2. rax contains f's address
    // 3. rbx contains guard's address
    naked_asm! {
        "push rbx",  // sp is incremented by 8
        "call rax",  // sp is incremented by 8 and now it is aligned again at the f invocation
        // after return, it is aligned by 8
        "ret",       // after returning to the `guard`, it is aligned by 16 again
    };
}
```

{{< callout title="Homework 3.1" >}}
Write a version that works on both architectures.
{{< /callout >}}

## Disclaimer

This text was written by Ivan Boldyrev. I used AI only for proofreading, but its
help was invaluable.

<!-- Local Variables: -->
<!-- spell-fu-buffer-session-localwords: ("coroutine" "coroutines" "MacOS" "stackful" "callee") -->
<!-- End: -->
