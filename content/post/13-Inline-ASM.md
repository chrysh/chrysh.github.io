---
title: Inline Assembly
date: '2026-04-22'
categories:
  - Rust
  - C
  - Linux Kernel
tags:
  - Rust
  - C
  - Linux Kernel
  - Assembly
---

[<img class="penguin" style="float: left; padding-left: 0%" src="/static/img/rusty_penguin_14.jpeg" alt="Rusty penguin. Created by DALL·E 3."/>](https://github.com/Rust-for-Linux/)

# Why inline assembly?

If you ever need to do something that's not possible in your high-level language,
or you need that extra bit of performance, inline assembly is your portal to the
processor. In C, the `asm` keyword is a direct portal to the processor. The
compiler treats your assembly string like a mysterious `__black box__`. You tell the
compiler, "I'm going to mess with some registers, don't worry about it," and C
simply hopes you know what you're doing. It's low-level, it's fast, and it's
remarkably easy to forgot to tell the compiler you clobbered the `eax`
register and now you find yourself [tackled by a raptor](https://xkcd.com/292/).

Like in many other places in C, also in the realm of assembly code, the
relationship between compiler and the asm block is based on `__trust__`. If
you forget to list a register in your clobber [^1] list, the compiler won't
tell you, it will just generate broken code that just crashes.

Rust elevates this relationship again to a `__formal contract__`. In Rust, even
assembly code is (somewhat) type aware. If you pass a 64-bit pointer into a
32-bit register operand, the compiler will raise a flag. By using the `label`
operand, the borrow checker and control flow graph is aware of where the code might
jump.

# A side-by-side comparison

To demonstrate how the readability improves when using Rust compared to C,
let's look at the following code [reading the time stamp
counter](https://www.felixcloutier.com/x86/rdtsc):

```C
#include <stdint.h>

uint64_t get_rdtsc(void) {
    uint32_t lo, hi;
    // Positional syntax: template : outputs : inputs
    __asm__ __volatile__ ("rdtsc" : "=a"(lo), "=d"(hi));
    return ((uint64_t)hi << 32) | lo;
}
```

In comparison, this is the corresponding Rust code:

```rust
pub fn get_rdtsc() -> u64 {
    let (lo, hi): (u32, u32);
    unsafe {
        asm!(
            "rdtsc",
            out("eax") lo,
            out("edx") hi,
            options(nomem, nostack, preserves_flags)
        );
    }
    ((hi as u64) << 32) | (lo as u64)
}
```

While in C the colons are positional (`__asm__("..." : output : input : clobbers);`),
in Rust the arguments can be positional or named. For the registers, C asm uses
single-letter codes (e.g. `=a` for output into eax), Rust uses explicit names
or classes (`in(reg)`, `out("eax")`) which makes it easier to read. Furthermore,
Rust handles clobbers automatically via `in(reg)`, `out(reg)`, or via options
like `nomem`.

## asm! Clobbering Options

| Option          | Effect on compiler optimization                                                                                                  |
| --------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| nomem           | Doesn't access memory, lets compiler cache variables in registers                                                                |
| nostack [^2]    | Doesn't modify the stack pointer, doesn't write to red-zone                                                                      |
| preserves_flags | Doesn't change CPU flags (like zero, overflow, carry), so compiler can skip recomputing condition flags                          |
| noreturn        | Assembly never returns (e.g.jmp or exit syscall, which never execute the   instruction `return`)                                 |
| readonly        | Doesn't write to memory. Only reads.                                                                                             |
| pure            | No side effects, outputs depend only on inputs. Allows compiler to eliminate/reorder.     Must use with nomem or readonly.       |
| raw             | Treats the assembly string as a literal, disabling Rust's string interpolation (allowing you to use `{}` without escaping them). |

For an extensive list, check out the [Inline Assembly Rust documentation](https://doc.rust-lang.org/reference/inline-assembly.html#syntax).

Furthermore, [ABI clobbers](https://doc.rust-lang.org/reference/inline-assembly.html#abi-clobbers) can be used to apply a default set of clobbers, for example, for a specific coding convention like `"C"`, `"sysv64"`, `"win64"`, etc. Generic register class outputs are not allowed when using this keyword.

# The Good, ..

While Rust's inline assembly was inspired by C's asm (and `__asm__`), several
defaults were changed to prioritize `__safety, readability, and modern
optimization__`.
While in Rust, the `asm!` macro is `__parsed by the compiler__`, in C, it is
forwarded to the assembler.

While C/GCC/Clang defaults to AT&T syntax (e.g., `movl %eax, %ebx`), which
is often considered harder to read due to the pervasive percent signs and
reversed operand order, Rust defaults to `__Intel syntax__` (e.g., `mov ebx,
eax`). This matches the documentation for most modern processors (x86/x86_64)
and is generally more concise.

By default, C assumes very little. You must explicitly list the cc (condition
codes) in the clobber list if you modify flags. Rust, on the other hand,
defaults to `__conservative clobbering__`. Rust assumes that condition flags
are modified unless you explicitly pass `options(preserves_flags)`. This prevents
subtle bugs where a compiler-generated branch is broken by assembly that
silently flipped a bit.

One of my main concerns with C asm is that it uses cryptic single-character
constraints (e.g., "r", "a", "m", "0"). Rust uses `__named register classes__`
and explicit argument binding. Instead of positional arguments (%0, %1), and
encourages `__named arguments__`, which makes assembly code at least
slightly more readable.

If a C asm block has no outputs, it is often treated as volatile
automatically, but memory side effects are generally assumed unless specified
otherwise. In Rust, the compiler assumes the assembly can modify memory
unless you explicitly provide `options(nomem)` or `options(readonly)`.
However, unlike C's `volatile`, Rust's asm! is `__"volatile" by default__` (it
won't be deleted unless marked `pure`).

Next on the list is that Rust introduces the concept of `__Register Classes__`.
Instead of forcing you to pick a specific register or use a cryptic C
constraint letter, you can tell Rust to "pick any general-purpose register."
This allows the `__LLVM register allocator__` to be more efficient, as it can
pick a register that is already free rather than being forced into a specific one
you hard-coded.

| Class   | Description                                           |
| ------- | ----------------------------------------------------- |
| reg     | Any general-purpose register (e.g., rax, rbx on x86). |
| xmm_reg | SSE registers on x86.                                 |
| kreg    | AVX-512 mask registers.                               |

For an extensive list of supported register classes, see the [official Rust inline assembly documentation](https://doc.rust-lang.org/reference/inline-assembly.html).

In C, if you use a register like rbx inside your assembly, you must manually add it
to a clobber list at the end of the block so the compiler knows to save its
value. In Rust, when you use `in`, `out`, or `inout` operands, the compiler
`__automatically handles the clobbering__` for those registers. You only
need to manually clobber a register if you use it without declaring it as
an operand.

By using `options(pure, nomem, preserves_flags)`, you provide the Rust
compiler with a "contract." If you tell the compiler the code is `pure`,
it can actually move your assembly block out of a loop or eliminate it
entirely if the result isn't used. Optimizations that are nearly impossible
to perform safely with C's more opaque asm blocks become possible in Rust.

Last but not least, Rust allows you to `__pass function symbols or static
variables__` directly into assembly using the sym keyword. This makes calling
Rust functions from within an assembly block much more straightforward than
calculating offsets or addresses manually in C, and less error prone than using
calculated values.

# ..the Bad, ..

But where there is light, there is shadow. Using Rust's `asm!` macro also has risks
and disadvantages.

**Disclaimer: The mentioned disadvantages are what I found while working with inline
assembly. Your mileage may vary.**

Any use of this macro has to be wrapped in an `unsafe` block. This can pollute an
otherwise clean codebase, where the Rust compiler usually handles memory
safety automatically.

While in modern gcc versions, an asm block can directly jump to a C label, Rust's
`asm!` is strictly local, which makes cross-language control flow more restricted.
Even though I would argue that jumping to labels and creating spaghetti code is not
a sign of a great software architecture anyways.

Last but not least, Rust's `asm!` is ultimately lowered to LLVM inline
assembly, which can optimize in an unexpected way. For example, if the
`options(..)` are slightly incorrect, the compiler might optimize away code or
might fail to inline a function. This can lead to silent errors where your
program might work fine in `Debug` mode, but crash or produce impossible
results in `Release` mode because the optimizer took your incorrect `option` at
face value.

# .. and the Ugly!

Other things to keep in mind is that inline assembly is "invisible" to many of
Rust's standard tools. The auto formatter `cargo fmt` does not reformat the
assembly code string, and the linter `clippy` cannot catch logical errors or
non-idiomatic assembly patterns.

When refactoring, silent errors can occur. Like in C asm, if input or output arguments
are swapped in the register list, the assembly block still executes, but performs
the operation on the wrong data. And since the assembly, as mentioned, is in
an `unsafe` block, the compiler cannot catch these errors.

Furthermore, if during refactoring a variable type is changed from `u32` to
`u64`, but your assembly code explicitly uses 32-bit registers, the
instruction will operate only on the lower bits and silently leave the upper
32 bits of your new `u64` variable as garbage data.

Last but not least, if you use named arguments in `asm!`, and you refactor your
surrounding Rust code and introduce a variable with the same or similar name,
you might end up accidentally binding the wrong variable.

To keep errors even between refactoring passes to a minimum, use `__named
arguments__` instead of positional arguments, keep blocks small and therefore
easier to audit, use `core::mem::offset_of!` if you access struct fields instead
of hard-coded numbers, and use `__unit tests__` to catch logic errors in
assembly early.

# Sub-register arguments and sizing

Here an example for adding x86_64 assembly to Rust code:

```rust
use std::arch::asm;

struct KernelHeader {
  id: u32,
  version: u16,
}

fn main() {
    let header = KernelHeader {id: 1, version: 10};

    unsafe {
        asm!(
            "mov eax, {val_i}",
            "mov eax, {val_v}",
            val_i = in(reg) header.id,
            val_v = in(reg) header.version,
        );
    }
}
```

Compiling this example gives us the following error for `val_i`:

```bash
warning: formatting may not be suitable for sub-register argument
  --> asm.rs:15:23
   |
15 |             "mov eax, {val_i}",
   |                       ^^^^^^^
...
19 |             val_i = in(reg) header.id,
   |                             --------- for this argument
   |
   = help: use `{0:e}` to have the register formatted as `eax` (for 32-bit values)
   = help: or use `{0:r}` to keep the default formatting of `rax` (for 64-bit values)
   = note: `#[warn(asm_sub_register)]` on by default
...
error: invalid operand for instruction
  --> asm.rs:15:14
   |
15 |             "mov eax, {val_i}",
   |              ^^^^^^^^^^^^^^^
   |
note: instantiated into assembly here
  --> <inline asm>:2:2
   |
 2 |         mov eax, rax
   |         ^
```

In x86_64 assembly, the register `al` denominates an `__8-bit register__`,
`ax` a `__16-bit__` register__, `eax` is `__32-bit__` and `rax` is a
`__64-bit register__`. The compiler warns us that we are trying to move a 64-bit
value into a 32-bit register (`eax`). Because our code is running on a 64-bit
system, the reg class, and therefore `{val_i}`, defaults to a 64-bit register.

And the Rust compiler also gives us an indication how to fix it: Using the
`{val_i:e}` modifier we tell the compiler "I know this should be a 64-bit
register, but I want you to use a 32-bit register explicitly".

While the C compiler treats the assembly string as a black box, the Rust
compiler `__parses and validates__` the assembly before it ever reaches the
assembler. The Rust language is more verbose and forces the developer to
write out things that would be undefined behavior in other languages
explicitly.

If you want your code to be even more portable to different architectures,
you can let the compiler do the mapping for you for all the arguments:

```rust
let mut out_i: u32;
let mut out_v: u32;

asm!(
    "mov {tmp_i:e}, {val_i:e}",
    "mov {tmp_v:e}, {val_v:e}",
    val_i = in(reg) header.id,
    val_v = in(reg) header.version,
    tmp_i = out(reg) out_i,
    tmp_v = out(reg) out_v,
);
```

If you absolutely must use `eax` because of specific hardware requirements
or an ABI, you can bind the variable directly to that register.

```rust
asm!(
    "/* eax is now bound to header.version */",
    "mov ebx, eax", 
    in("eax") header.version as u32, // Cast to 32-bit to match 'eax' width
    out("ebx") _,                    // Clobber ebx
);
```

# Conclusion

Rust's asm! macro is undeniably a triumph of ergonomics. It replaces C's
cryptic hieroglyphics with named arguments and Intel syntax, making
low-level code look almost... civilized. It's the "polite" way to tell the
CPU exactly what to do. But once you step inside that unsafe block, it's
still a minefield where you are only one typo away from a segfault that will
take days to debug. While in C, you can shoot yourself in the foot, in Rust
you first sign a waiver before shooting yourself in the foot.

---

[^1]: If, like me, you don't know what clobber is, it happens when you
unintentionally overwrite data. When writing inline assembly in C, you
may need to specify a "clobber list" (or "clobbered register list") to tell
the compiler which registers are modified, preventing the compiler from
using them for other tasks.

[^2]:  The red zone is a 128-byte area below the stack pointer in x86-64 System V
ABI that functions can use for temporary storage without adjusting the stack
pointer. It is located at `$rsp - 128` to `$rsp - 8` and reserved for
the current function. Therefore, interrupt handlers and signal handlers
`__won't clobber it__`. The function `__must__` be a `__leaf function__`
(i.e. not call any other function). A leaf function may directly use `[rsp-8]`
without performing `sub $8, %rsp` / `add $8, %rsp`. The `nostack` option
tells the compiler
