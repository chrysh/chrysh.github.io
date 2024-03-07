---
title: Allocators
date: '2024-02-25'
categories:
  - Rust
  - Kernel Driver
  - Linux Kernel
  - Allocators
tags:
  - Rust
  - Kernel Driver
  - Linux Kernel
  - Allocators
  - std crate
---

[<img src="/static/img/rusty_penguin_6.jpeg" style="max-width:40%;min-width:40px;float:right;padding:40px" alt="Rusty penguin. Created by DALL·E 3." />](https://github.com/Rust-for-Linux/)

# User Land
## When is it used?
In user land, the programmer typically does not explicitly call an allocator.
Instead, whenever you use types like `Box`, `Vec` or `String`, in the **background**
the **global allocator** is used to allocate and free memory. This global allocator
is part of the Rust standard library `std`. Therefore, in order to define your
own allocator, you have to import the GlobalAlloc trait and implement it:

```
use std::alloc::GlobalAlloc
```

For userspace programs, the standard allocator used can be found in
`library/alloc/src/alloc.rs`. Per default, if no **custom allocator** is defined to
be used, the allocation call results in a call to `System.alloc`. The allocator
used for the system can be hunted down through many abstraction layer to the
[libc crate](https://github.com/rust-lang/libc), where the bindings for **each
operating system** are implemented.

## How to add an allocator

The trait `std::alloc::GlobalAlloc` defines the **interface** for global memory
allocators. The Rust compiler plugs in the memory allocator that is actually
used at compile time. If you want to add your own custom allocator, you have to
implement this trait and, using **attribute** `#[global_allocator]`, tell the
Rust compiler to use this allocator instead of the default.

```
use std::alloc::{GlobalAlloc, Layout};

struct MyAllocator;

unsafe impl GlobalAlloc for KernelAllocator {
    fn alloc(&self, layout: Layout) -> *mut u8 {
        // Add memory and return a pointer to it
    }

    fn dealloc(&self, ptr: *mut u8, _layout: Layout) {
        // Free memory and return it to memory pool
    }
}

#[global_allocator]
static ALLOCATOR: MyAllocator = MyAllocator;
```

`struct Layout` contains information about the size of memory to allocate as
well as the **alignment** requirement of the memory (which always need to be a
power of two).

Writing your own memory allocator is useful only in certain situations, when you
need **fine-grained control** over memory management. This can be the case when
writing **low level code** for an embedded system, or an **operating system**,
which brings us to the next chapter.

# Kernel Land

## Differences to User Land

One obvious differences to the userspace memory allocator is that the kernel
memory allocator is operating in **kernel space**. Moreover, while the Rust
standard library uses the **system's memory allocation APIs** (e.g. `malloc` and
`free` on Unix-like systems). In the Linux kernel space on the other hand, the
**kernel allocators** are used (e.g. krealloc, kfree).

Instead of the **std crate**, which relies on underlying operating system services
such as threads, files, networks, heap memory allocation, the kernel uses the
**core crate**.
```
use core::alloc::GlobalAlloc
```

If you look at the difference in the `alloc/` directory between [Rust for
Linux](https://github.com/Rust-for-Linux/linux/blob/rust-next/rust]) and
[rust-lang](https://github.com/rust-lang/rust), you won't see many differences.
Most of the kernel specific allocation code you will find in the file
`rust/kernel/allocator.rs`. The part that already made it **mainline** implements
`alloc` and `dealloc`, which a person writing modules will not come in contact with
anyways. More changes that might eventually be integrated can be found in the
[Rust-for-Linux](https://github.com/Rust-for-Linux/) repository. Those changes
would use `krealloc_aligned` in order to prevent **misaligned allocations**,
because the alignment guarantees provided by the SLAB allocator might be smaller
than the ones required by the `struct Layout`. Unaligned memory access can lead
to segmentation faults on some architectures, and slow down memory access on
others.

The functions implemented use
[bindings](https://blog.christina-quast.de/post/2024/02/02/creating-c-bindings/)
to call the kernel alloc functions. The only gotcha is that you have to call
krealloc instead of the usually used `kmalloc` function, because kmalloc is an
inline function and can therefore not be **bound to a result**.  In other words,
bindings can only work on functions over a **FFI** (foreign function interface).
But calling `krealloc` with a **null pointern** has the same effect as a call to
`kmalloc` would have.

The function furthermore has to be wrapped in an `unsafe {}` block, because the
compiler can not **analyze C code** and **assure memory safety** of that code,
etc. The unsafe block tells the compiler not to bother analyzing this part of
the code.
