---
title: Creating C bindings
date: '2024-02-02'
categories:
  - Rust
  - Kernel Driver
  - Linux Kernel
  - C bindings
tags:
  - Markdown
---

# What are bindings?
Bindings are ways to create an interface between two programming languages,
which allow code written in one language to call from code in another language.
In the case of Rust in the Linux kernel code, **Rust FFI (Foreign Function
Interface)** bindings are used to call C functions, conforming to the C calling
convention.  This means that arguments are placed on the stack or in registers,
as the C function is expecting to find them there, and cleans up after the
function call. Furthermore, the FFI takes care of converting data types between
the two languages and handles memory management.  The calling conventions are
dependent on the architecture used (arm, arm64, riscv, x86,..), because each
architecture has a different **calling convention** and expects the argument in
different places (the stack, registers, etc). And even between different
compilers or compiler versions, the calling conventions can differ! The bindings
abstract away those differences.

[<img src="/static/img/rusty_penguin_3.jpeg" style="max-width:35%;min-width:40px;float:right;padding:50px" alt="Rusty penguin. Created by DALL·E 3." />](https://github.com/Rust-for-Linux/)

# How bindings are created

`bindgen` uses `libclang` to parse C/C++ header files. After parsing, `bindgen`
generates Rust code than can be used to call those functions, and Rust structs,
that correspond to the C structs. Furthermore, it also creates enums, typedefs,
etc. In userspace C code, you can include the generated file `bindings.rs` in
order to access those C language constructs from your Rust program.

In reference to the Linux Kernel, you call `make` to compile your kernel, the
bindings are created for you.  If you change some bindings, and call `make
modules` without recompiling the kernel, the system might complain that it
cannot find some symbols. The reason for this is that `bindgen` will not have
generated the symbols for the bindings yet, because it only recompiled your
kernel module, but not the whole kernel. Try calling `make` again, or even
better, fix the underlying dependency issue.

# How to add bindings

**Disclaimer: The mentioned ways is what I found in
[Rust-for-Linux](https://github.com/Rust-for-Linux/). It is possible that
mainline will use a slightly different way**

C functions can be called with the syntax `bindings::my_func(my_args)`. This
call has to be wrapped in an `unsafe{}` block, because any call to a C function
is deemed unsafe and therefore you as a caller should that do all necessary
checks so that no undefined behavior occurs. The convention is to describe in a
`// SAFETY` comment why it is safe to call this function.

```
    pub fn genphy_config_aneg(&mut self) -> Result {
        let phydev = self.0.get();
        // SAFETY: `phydev` is pointing to a valid object by the type invariant of `Self`.
        // So it's just an FFI call.
        // second param = false => autoneg not requested
        to_result(unsafe { bindings::__genphy_config_aneg(phydev, false) })
    }
```

Bindings exist not only for **functions**, but also for **structs** and **variables** that
you want to use in your Rust code. If on the other hand you want to use a macro
instead of a function, you need to add use a different technique: You have to
add a **wrapper function** manually into the `rust/bindings/bindings_helper.h`
file, because macros are not part of the C language's binary interface:

```
+unsigned int rust_helper_major(dev_t dev) {
+    return MAJOR(dev);
+}
+EXPORT_SYMBOL_GPL(rust_helper_major);
+
```

**Bindgen** removes the `rust_helper_` part of the string, so that you can use
this Rust function by calling `bindings::major(...)`.

# Debug when binding symbols go missing

Rust seems to be using a similar name mangling scheme as C++ for encoding
functions names, so that if you find a cryptic function name, you can use
`c++filt` in order to decode them:

```
% c++filt
_RNvXs_Cs2WjrwJ2NH7t_13rockchip_rustNtB4_11PhyRockchipNtNtNtCsjDtqRIL3JAG_6kernel3net3phy6Driver7suspend

<rockchip_rust[22401484ae8718f5]::PhyRockchip as kernel[e4b87d7aec0e5630]::net::phy::Driver>::suspend
```

The mangling could be different in some situations, but so far, it has worked
for me. The mangled function name include information about generic parameters,
lifetimes, function parameters, etc. Functions used with `extern "C" keyword are
not mangled, since they are already using the C calling convention.

You can find the symbols of the symbol that are compiled into the kernel `vmlinux` through
the use of `objdump -t ../vmlinux.o` (or use string, the poor man's `objdump`).
If your function name is not there, the linker will throw an error while
compiling the module, and you need to call `make` again to recompile the kernel.
If your symbol is still not there, you need to start proper to debugging to see
why the compiler considers your function dead code. Or search for spelling
mistakes in your functions by comparing the symbol needed to the symbol that is
exported!
