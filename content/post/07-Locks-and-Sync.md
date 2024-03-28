---
title: Locks and synchronization
date: '2024-03-10'
categories:
  - Rust
  - C
  - Kernel Driver
  - Linux Kernel
tags:
  - Rust
  - Kernel Driver
  - Linux Kernel
---

[<img class="penguin" src="/static/img/rusty_penguin_8.jpeg" alt="Rusty penguin. Created by DALL·E 3." />](https://github.com/Rust-for-Linux/)

# Synchronization

The [LWN article](https://lwn.net/Articles/863459/) was an attempt to give a
side by side code comparison of how the same PL061 GPIO driver looks like
written in C and Rust. Even though the synchronization code that ended up in the
Linux Kernel is different from what we see there, it gives a good overview of
how a driver using locking primitives could look like. For this blog post, we
will focus on the locking and synchronization mechanisms used by C and Rust,
that can be found in the latest Linux kernels.

For a more general explanation focusing on how to port a driver to Rust, check
out this [blog post](../../../../2024/02/06/porting-my-first-phy-driver/).

Before deep diving into how locks are implemented in the Linux kernel using
Rust, let us start with the general principle of locking in Rust. Rust programs
are inherently memory and thread safe, given that the compiler assures at
compile time that data can not be read and modified at the same time. Similar to
the principle of [rwlocks](https://www.unix.com/man-page/linux/3c/rwlock/),
which are multiple reader single writer locks, the Rust compiler makes sure that
either multiple readers can have a non-mutable reference to a data element, or
exactly one writer has a mutable reference at the same time. The mutable
references have to be marked explicitly with the keyword `mut` in the code.
Nevertheless, there are some specifics to using locks in the kernel, which we
will dive into next.

The kernel APIs related to synchronization that have been ported or wrapped for
usage by Rust code in the kernel can be found in `rust/kernel/sync.rs`. At the
current state, that are the `arc`, `condvar`, `lock` and `locked_by` crates.

For the time being, we concentrate on the `lock` crate, where the types
`mutex::Mutex` and `spinlock::SpinLock` are implemented.

## Spinlocks

The principle of a spinlock is, as the name suggest, when multiple CPUs attempt
to lock the same spinlock, only one is allowed to progress at a time, while the
other CPUs will continue spinning until the spinlock is unlocked. At this point,
the next CPU will be allowed to progress in its execution.

When looking at spinlock usage, the first apparent difference is the between
Rust and C drivers lies in the include files. The functions and data relevant
for spinlocks used in drivers written in C are located in `#include
<linux/spinlock.h>`. For Rust drivers, the necessary data structs are located in
`rust/kernel/sync/spinlock.rs`. The implementation in `spinlock.rs` is based the
kernel's `spinlock_t`.  Instances of `SpinLock` need a lock class and to be
pinned, which means that the data at a particular memory address will not and
cannot be moved. It is "pinned" to that location. This is particularly important
in async programming, because we often deal with "futures". A future may contain
references to its own data. If the future were to be moved to a different memory
location, the reference would not be valid anymore, which would lead to
undefined behavior.

The recommended way use spinlocks is therefore through the usage of the
`pin_init` and `new_spinlock` macros.  If you wanted to declare, allocate and
initialize a struct `Example`, this is how you would code it:

```
use kernel::{init::InPlaceInit, init::PinInit, new_spinlock, pin_init, sync::SpinLock};

struct Inner {
    a: u32,
    b: u32,
}

#[pin_data]
struct Example {
    c: u32,
    #[pin]
    d: SpinLock<Inner>,
}

impl Example {
    fn new() -> impl PinInit<Self> {
        pin_init!(Self {
            c: 10,
            d <- new_spinlock!(Inner { a: 20, b: 30 }),
        })
    }
}

// Allocate a boxed `Example`.
let e = Box::pin_init(Example::new())?;
assert_eq!(e.c, 10);
assert_eq!(e.d.lock().a, 20);
assert_eq!(e.d.lock().b, 30);
```
`new_spinlock` is a macro that creates a spinlock and initializes it with the
given name and newly-created lock class. If no name is given, it generates one
based on the file name and line number.

```
#[macro_export]
macro_rules! new_spinlock {
    ($inner:expr $(, $name:literal)? $(,)?) => {
        $crate::sync::SpinLock::new(
            $inner, $crate::optional_name!($($name)?), $crate::static_lock_class!())
    };
}
```

We can also use interior mutability to modify the contents of a struct, despite
only having a shared reference.

```
use kernel::sync::SpinLock;

struct Example {
    a: u32,
    b: u32,
}

fn example(m: &SpinLock<Example>) {
    let mut guard = m.lock();
    guard.a += 10;
    guard.b += 20;
}
```
All these examples are taken from the `rust/kernel/sync/spinlock.rs` file.

The implementation is backed by the underlying kernel `spinlock_t` object, and
the functions `__spin_lock_init`, `spin_lock` and `spin_unlock` through FFI
binding, which ensures mutual exclusion.

```
unsafe impl super::Backend for SpinLockBackend {
    type State = bindings::spinlock_t;
    type GuardState = ();

    unsafe fn init(
        ptr: *mut Self::State,
        name: *const core::ffi::c_char,
        key: *mut bindings::lock_class_key,
    ) {
        // SAFETY: The safety requirements ensure that `ptr` is valid for writes, and `name` and
        // `key` are valid for read indefinitely.
        unsafe { bindings::__spin_lock_init(ptr, name, key) }
    }

    unsafe fn lock(ptr: *mut Self::State) -> Self::GuardState {
        // SAFETY: The safety requirements of this function ensure that `ptr` points to valid
        // memory, and that it has been initialised before.
        unsafe { bindings::spin_lock(ptr) }
    }

    unsafe fn unlock(ptr: *mut Self::State, _guard_state: &Self::GuardState) {
        // SAFETY: The safety requirements of this function ensure that `ptr` is valid and that the
        // caller is the owner of the mutex.
        unsafe { bindings::spin_unlock(ptr) }
    }
}
```

## Mutexes

The implementation of the mutexes is very similar to what we already covered in
the chapter on spinlocks. Instead of a `spinlock_t`, a `struct mutex` is used,
but everything else is the same. Since, compared to spinlocks, mutexes can
block, they should generally not be used in atomic contexts.

## Mainline vs Rust for Linux

The spinlock implementation in the [Rust for
Linux](https://github.com/Rust-for-Linux/) project adds an abstractions for both,
`spinlock_t` and `raw_spinlocks_t`. Raw spinlocks are guarantees not to sleep even on RT
variants of the kernel.

Furthermore, while the spinlock implementation described in this blog post can
be used in interrupt context, the programmer either needs to manually disable
and restore interrupts on the local processor, or use the functions
`spin_lock_irqsave` and `spin_unlock_irqrestore`, which take care of that. So
while the code presented is a good start, more work is needed to make more
Kernel spinlock functionality available to Rust.
