---
title: Opaque Pointers and Type Erasure
date: '2026-04-30'
categories:
  - Rust
  - Linux Kernel
tags:
  - Rust
  - Linux Kernel
  - FFI
  - opaque pointers
  - type erasure
  - safety
---

[<img class="penguin" style="float: right; padding-left: 5%" src="/static/img/rusty_penguin_16.jpeg" alt="Rusty penguin. Created by DALL-E 3."/>](https://github.com/Rust-for-Linux/)

In the last blog post, we discussed how callback arguments are handled in Rust when
interfacing with C.

When we want to interface with every C context in the Linux kernel using `*mut
ffi::c_void`, we lose the ability to __distinguish different types__. If we
interpret a device context as a driver context, in the best case we run into a
segfault. In the worst case, we hunt a bug that only sometimes overwrites our
whole file system.

Let's have a look at rust/library/core/src/marker.rs, where `PhantomData` is defined.

```rust
/// ## Layout
///
/// For all `T`, the following are guaranteed:
/// * `size_of::<PhantomData<T>>() == 0`
/// * `align_of::<PhantomData<T>>() == 1`
///
/// [drop check]: Drop#drop-check
#[lang = "phantom_data"]
#[stable(feature = "rust1", since = "1.0.0")]
pub struct PhantomData<T: PointeeSized>;
```

The compiler uses `PhantomData<T>` to find out the __lifetime and ownership__
(variance) of a generic type. It tells the compiler to treat a struct as if it
owns a `T` field, even if it does not actually store one, making sure the
__C data is not dropped__ while the Rust struct is still in use. When
PhantomData is included in your struct, the pointer cannot automatically be
marked as `Send` or `Sync` if the underlying C library is not thread-safe.

```rust
#[stable(feature = "rust1", since = "1.0.0")]
impl<T: PointeeSized> !Send for *const T {}
#[stable(feature = "rust1", since = "1.0.0")]
impl<T: PointeeSized> !Send for *mut T {}
```

In theory, you could replace `PhantomData` with a private field `_unused: [u8;
0]`. However, you would lose the automatic inheritance of the thread-safety
properties of `T`. We want a type that allows a non-thread-safe library to be
marked as __not thread-safe by default__. Furthermore, if `T` has a destructor,
using `PhantomData<T>` prevents the compiler's "drop checker" from destroying
the context while the data is still supposedly in use.

The attribute `#[lang = "phantom_data"]` also marks this struct as a language
item, which the compiler needs to know about in order to implement core language
features. It provides a "hook" between the library and the compiler.
The size of PhantomData is 0, which means it does not take up space in memory.
The compiler ensures that `PhantomData` is a zero-sized type (ZST) and is completely
optimized away in the final machine code.

`PhantomData<T>` makes it possible to introduce __type safety__ while the
details of the implementation of the type are still only known to the C library
(__type erasure__). 

```rust
struct DeviceHandle {
    ptr: *mut c_void,
    _marker: PhantomData<DeviceContext>,
}

struct DriverHandle {
    ptr: *mut c_void,
    _marker: PhantomData<DriverContext>,
}

// The compiler treats these as totally different entities:
fn configure_device(handle: DeviceHandle) { ... }

// This call will fail if you pass a DriverHandle:
// configure_device(my_driver_handle);
```

Even if both structs might be zero-sized or contain a raw pointer type, the `T`
inside of `PhantomData<T>` makes them __incompatible__ in the eyes of the
type system. If you now want to pass a struct containing `DeviceHandle` to a
function that expects a `DriverHandle`, the Rust compiler will yell at you. 

`T` has the trait `PointeeSized`, which tells the compiler that the size of the data
is only known at runtime via a pointer. In contrast to this is the `Sized` trait, where
the size is known at compile-time. We have to use `PointeeSized`, because at compile time
we cannot know the size of the struct in the C code.[^1]

Furthermore, it implements several constructs for all traits that might want to be used
in the kernel, like `Hash`, several versions of `Eq` (for testing equality) and
`Ord` (for testing smaller than, etc), `Copy`, and `Default`, so that we don't have to.
```rust
#[stable(feature = "rust1", since = "1.0.0")]
impl<T: PointeeSized> Hash for PhantomData<T> {
    #[inline]
    fn hash<H: Hasher>(&self, _: &mut H) {}
}

#[stable(feature = "rust1", since = "1.0.0")]
impl<T: PointeeSized> cmp::PartialEq for PhantomData<T> {
    fn eq(&self, _other: &PhantomData<T>) -> bool {
        true
    }
}
```

The second noteworthy construct is `Opaque<T>`. `Opaque` marks FFI objects that
are __never interpreted by Rust__ code. It is used for __wrapping structs__ from the C
side, and gets rid of all usual assumptions Rust has for a value. So the value
may be uninitialized, it is possible that multiple shared references
`&Opaque<T>` exist (while the data it encapsulates can still be modified through
__internal mutation__ using `UnsafeCell`), and the value shall never be shared
with other threads (i.e. it is `!Sync`). This struct has to be used
whenever the C side has access to the value, since it can't be ensured that the
C side is adhering to the usual constraints and safeguards that Rust needs.

```rust
#[repr(transparent)]
pub struct Opaque<T> {
    value: UnsafeCell<MaybeUninit<T>>,
    _pin: PhantomPinned,
}
```

This code defines Opaque as a __safe wrapper for unsafe memory__. It
occupies the same space as the C type `T` (`#[repr(transparent)]`),
can be modified by external C code without the Rust compiler yelling (using `UnsafeCell`),
tells the compiler that the data might be invalid (`MaybeUninit`) on creation and makes sure
that once the struct is created, it stays at a fixed address in memory to prevent
pointer corruption.

Let's have a look at a concrete example where Opaque is used in `rust/kernel/iov.rs`.

```rust
#[repr(transparent)]
pub struct IovIterDest<'data> {
    iov: Opaque<bindings::iov_iter>,
    /// Represent to the type system that this value contains a pointer to readable data it does
    /// not own.
    _source: PhantomData<&'data [u8]>,
}
```

In this piece of code, several of Rust's safety mechanisms are at work, namely lifetime tracking
and null pointer safety, all while keeping the __same memory footprint__ as the C code would
have.

By using `'data` as the lifetime of this struct, if you create `IovIterDest` from
a buffer, the compiler forces that the buffer outlives `IovIterDest`, preventing __use-after-free
vulnerabilities__. By wrapping the struct `bindings::iov_iter`, you make sure that
`IovIterDest` is always constructed from a __valid and non-null reference__, effectively moving
the `null` check to the initialization.
Because `Opaque` is wrapping the `iov_iter`, it prevents you from modifying
fields manually. The programmer is forced to use the kernel's provided functions and therefore
the __C side's invariants__ are kept __intact__. 
And since the `PhantomData` and `Opaque` wrapper have __no runtime footprint__, the machine code
compiled from this struct is identical to a raw `struct iov_iter *`. You can have your cake and
eat it!


# TL;DR

Rust uses __zero-code abstractions__ like `Opaque<T>` and `PhantomData<T>` in order to perform
static code analysis. The `PhantomData<T>` marker informs the compiler about lifetimes, ownership
and variance for types it doesn't actually store at runtime. `Opaque<T>` is a wrapper
that marks a struct as being owned by the C code, and therefore prevents the Rust compiler from
making __unsafe assumptions__ about initialization, aliasing or thread-safety of the object.
Last but not least, by treating distinct C handles (like `DeviceHandle` vs `DriverHandle`)
as incompatible types, the compiler can guarantee __type safety__ in the Rust code even if
in the C code it boils down to a mere `*mut c_void`.

---

[^1]: In Rust, every pointer consists of the actual __data address__ and optional __metadata__.
`PointeeSized` is implemented by types where the size of the object can be determined at runtime
just by looking at the pointer's metadata. Examples for those dynamically sized types are `[u8]`
or `dyn MyTrait`. Their pointer metadata is `usize` for the vector length or a vtable pointer
for the dynamic trait. Statically Sized Types marked by the `Sized` marker have a size known at
compile-time (`i32`, `[u8; 64]`). Their metadata is simply `()`.
