---
title: C vs Rust - In the Linux Kernel
date: '2024-02-23'
categories:
  - Rust
  - C
  - Kernel Driver
  - Linux Kernel
  - Community
tags:
  - Rust vs C
  - Kernel Driver
  - Linux Kernel
  - Community
  - Memory safety
  - Buffer overflow
  - Error handling
  - Cleanup
  - Linker
  - Compiler
---

# +++ Plus +++

[<img class="penguin" src="/static/img/rusty_penguin_5.jpeg" alt="Rusty penguin. Created by DALL·E 3." />](https://github.com/Rust-for-Linux/)
## Memory bugs and Concurrency

Let's get the obvious out of the way: With Rust, you will inherently write safer
code, because the compiler will complain until you do. **Whole error classes**
will become less common when Rust programs are widely used:
**Buffer-Overflows**, many types of **Memory-Leaks, Race conditions** are made
impossible, because the compiler will force you to have only multiple readers
for shared data **OR** one writer at a time for mutable data. Well, unless you
wrap all your code in `unsafe` blocks, which bypasses some of the compiler
checks. Furthermore, using cyclic data structures can also lead to memory leaks
because they always have a reference count > 1 and can therefore not be cleaned
up.

## Error handling and Cleanup
When you look into any driver code, you will find there are a lot of constructs
with the following pattern: `if(ret < 0) { goto cleanup_A; }`. They render the
code **harder to read** and make it way too easy to forget to check the **return
value**. The Rust way of error handling is much more explicit: Rust functions
that can fail return a **Result type**, and the `?` operator can be used to
propagate those errors up the stack, which makes the overall code more
**readable**. The compiler will force you to **acknowledge the error** somehow.
So the equivalent of the following C code is much shorter in Rust:

```
# C
int mydriver_function() {
    int ret;

    ret = do_something_A();
    if (ret < 0)
        return ret;

    ret = do_something_B();
    if (ret < 0) {
        goto cleanup_A;
    }

cleanup_A:
    // Cleanup  A

    return 0;
}

# Rust
fn mydriver_function() {
    do_something_A()?;
    do_something_B()
}
```

Compared to out-of-band exception handling used in C++, where error handling is
a separate control flow mechanism from your code, exceptions in Rust are treated
**in-band**. This means that errors are just another type of return value, and
the compiler generates the code for checking return values for you. The
programmer can then actively decide to ignore the specific generated error with
calling functions like `unwrap()`, etc., which is discouraged in production code
for obvious reasons.

Furthermore, for each struct, Rust implements a `Drop()` function which **cleans
up** the allocated resources for you. Especially in the context of Kernel Drivers
this is useful, because you don't have to take care of calling free wherever the
last pointer to your allocated memory goes out of scope, similar to destructors
in C++. The compiler makes sure the call to the drop function happens
**automatically**.  This does not apply to unsafe code, where you have to
manually take care of freeing resources.

## Code size
Rewriting the rockchip driver in Rust resulted in a **reduction of around 10\%**
smaller code file `rockchip_rust.rs` compared to `rockchip.c` written in C. The
**lines of code** were reduced by stunning **50\%** in this instance.

## Outside of the Kernel
In C it is common practise to use many dynamically linked libraries, which in
turn results in a lot of **unused code** generated by the compiler. For example,
if you have a function pointer, in some cases the linker can not know that this
function is never called, and therefore has to generate code for it, which leads
to **bigger mapped code sections** in memory. Rust crates are usually statically
linked into the binary, which improves portability because the binaries do not
have runtime dependencies on shared libraries, and can therefore be distributed
more easily.

# \_-\_ Minus \_-\_
## Binary file size
While the size of the source code when you want to add your driver is smaller,
the resulting kernel module is still **bigger**. In the case of the rockchip phy
driver, the kernel module file resulting from C code was **approximately 15\%
smaller** than the Rust equivalent. This will probably still improve in the future
with advancements of the compiler and integration in the kernel. But as it stands
now, when using Rust instead of C one is paying with bigger binaries for the
safety advantages of Rust.
Generally speaking, Rust binary file sizes will usually be bigger than their C
equivalent, since Rust does not use **dynamic linking** and the **code duplication**
resulting from it, and furthermore because of the additional checks and
data structures the compiler generates.

## Performance
While I did not do any performance tests for the Rust code to present numbers,
in general performance of comparable C code is either comparable or still
**faster**, especially for **low-level** systems programming tasks.

## Outside of the Kernel
In contrast, the Rust compiler tries to **inline more code** and instantiates code
for **type parameter** to a generic type, similar to what C++ does. This results in a
log of **duplicate instructions** in the machine code, leading to **bigger
binary sizes**.  This is is also why Rust can not use **dynamic libraries**, except for
when using a C API to link against a C library. But using C code together with Rust
defeats the **safety promises** Rust makes, since a buffer overflow can still happen
in the C library function called by your Rust code.

The Linux kernel does not use **cargo** to add code to your Kernel drivers, and
never will. But using cargo is a curse and a blessing. On the bright side, it
makes it easy to use external libraries independent of your operating system,
you just call `cargo add external_lib`, and the correct version is added to your
Cargo.toml file. On negative aspect of that is, that your program and your
program's dependencies might depend on **different versions** of this package, and
you have to maintain security updates across all the versions.
