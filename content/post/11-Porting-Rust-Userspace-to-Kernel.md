---
title: Porting a userspace program to Kernel space
date: '2024-08-09'
categories:
  - Rust
  - C
  - Kernel Driver
  - Linux Kernel
  - Userspace
tags:
  - Rust
  - Kernel Driver
  - Linux Kernel
  - Quarto
---

[<img class="penguin" style="float: left; padding-left: 0%" src="/static/img/rusty_penguin_12.jpeg" alt="Rusty penguin. Created by DALL·E 3." />](https://github.com/Rust-for-Linux/)

# Differences between Kernel and Userspace
Porting a Rust program to the Linux Kernel is very similar to porting a Rust
program to an embedded device. Compared to the program you write on your desktop
computer, you cannot use the standard library (`std`). Furthermore, embedded
systems and the Linux Kernel have special requirements for memory management.
Allocations in Kernel space must not fail, because this would lead to a kernel
panic, that makes the whole system crash. And we don't want our operating system
to crash.

So while it might seem easy on a first glance to port the program, let's dig
deeper into the pitfalls and problems you will encounter.

## TL;DR

If you ever wanted to port  a userspace Rust program to the kernel, you would
need to take care of the following things:

1. Change imports of `std::` lib to use `kernel::` or `core::`. You can not use
  any crates without porting them. Even `Vec` has a different implementation in
the kernel space.

2. You cannot use the command line for input/output. User interaction can
  happen through other communication interfaces, e.g.
read/write of a character device, or the `/sys` interface

3. Allocations in the kernel can fail when it is low on memory. Therefore, you
   have to implement and work with fallible allocations (e.g. `collect` would
turn into `try_collect` for the `Vec` type, see
[rust-lang](https://github.com/rust-lang/rust/issues/94047))

4. The [Rust-for-Linux](https://github.com/Rust-for-Linux/) project implemented
  the allocator for you, so at least, you don't need to take care that.

5. All occurrences of `println!` and other print-related macros need to be
   replaced with the kernel equivalents `pr_info!`, `pr_warn!`, `pr_err`, etc.

6. To interact with the system, instead of using `std::time::`, the kernel
   equivalent has to be used, and implemented if it does not exist yet. It might
be necessary to add [bindings](../../../../2024/02/02/creating-c-bindings/) to C
Kernel functions to use them.

7. You have to add a Kernel Makefile if you want to compile your Rust kernel
   module as an out-of-tree kernel module for your specific kernel. If you want
to be able to select your module with `make menuconfig`, you will also have to
add an entry to the `KConfig` file.

And this is not an exhaustive list.

## There is no stdlib

Besides changing the imports from `std::` to `kernel::` or `core::`,
you also [cannot use the stdlib in kernel space](../../../02/25/allocators/),
then [adding a few C bindings](../../../02/02/creating-c-bindings/) to interact
with existing kernel code, and making sure that all the functions you use should
not panic and crash the rest of the kernel, the code should globally be the
same. This blog entry gives you an idea how complex the work actually is.

## There is no stdin/stdout
When you start a program in bash, you usually communicate with the program by
writing to `stdout` (file descriptor 1) and reading user input from `stdin` (file
descriptor 0). Those file descriptors are provided to you by the kernel. You can
still pass and receive data from/to the kernel through other mechanisms. For
example, if you register a character device driver, you can implement the
read/write function associated with the device driver to pass data. You are
using read/write every time you do an echo into a `/dev/tty`.

Furthermore, the kernel provides a means to configure drivers or inform you about
stats through `/sys` and `/proc` file systems. This could be used as the
`stdout` of the game as well, although the solution with the character device is
surely much cleaner.

## There is no `collect`
The [Rust kernel](https://github.com/Rust-for-Linux/) copies over the current
content of [rust-lang](https://github.com/rust-lang/rust) into the `rust`
directory, and adds kernel specific code to `rust/kernel`. Not all functions can
be used in the kernel though. You don't want to use functions that could panic
in the kernel because then the whole system will panic. Therefore, you cannot
just use the function `collect` to create a Vector, you would have to use
`try_collect` - which is [not implemented
yet](https://github.com/rust-lang/rust/issues/94047).

## There are no crates

Oftentimes, userspace Rust programs will use crates. My example program uses
HashSet, for which there is no kernel equivalent yet. There are some lightweight
crates that are useful for embedded systems. And even comparatively small crates
often times have dependencies on other crates.
