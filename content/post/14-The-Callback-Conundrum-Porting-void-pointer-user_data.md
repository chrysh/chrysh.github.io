---
title: The Callback Conundrum - Porting void pointers to user data
date: '2026-04-29'
categories:
  - Rust
  - Linux Kernel
tags:
  - Rust
  - Linux Kernel
  - FFI
  - callbacks
  - platform drivers
---

[<img class="penguin" style="float: right; padding-left: 5%" src="/static/img/rusty_penguin_15.jpeg" alt="Rusty penguin. Created by DALL·E 3."/>](https://github.com/Rust-for-Linux/)

Platform drivers were one of the first success stories of the Rust-for-Linux
project. They were a proof of concept that the integration of Rust into the
existing Linux kernel C code base would work. The decision to focus on platform
drivers early on was strategic. They are simpler compared to complex GPU or
network drivers, and therefore provide a perfect sandbox for testing Rust
abstractions.

Therefore, we want to show the callback conundrum based on the Rust implementation of
the platform driver base `rust/kernel/platform.rs` in Linux kernel Version 7.0
of Linux Torvald's tree.

# The function of `void *user_data`
Most C callback functions have the following signature:
`void register_callback(void (*cb)(int, void*), void* user_data);`. This
signature is a common C pattern to pass a __private state or context__ to a __stateless
function pointer__. You can imagine the `user_data` like a bucket of bytes, that
has no information for the calling function of the callback in C. The usage of
`user_data` is a way of handling the __lack of native polymorphism__ and
__inheritance__ in C.

The userdata is usually cast to a driver dependent struct, depending on which
driver callback was called, and contains the __private state__ of a specific
instance of a device. A single gpio controller driver can be managing several
identical pieces of hardware at once. The user data is the "memory" that tells
the driver which one of them it is currently talking to. Therefore, this user
data pointer often times points to information like the hardware resource
mapping [^1], state management [^2], back pointers to the parent device or
parent system [^3], as well as buffer and queues [^4].

Let's look at `drivers/platform/goldfish/goldfish_pipe.c` for a code example.
The goldfish driver provides a very fast communication channel between the
guest system and the QEMU emulator.
In the call of the open callback function for this device driver, the field `private_data`
is initialized with a pointer to the goldfish specific `struct goldfish_pipe`. When the
device disappears, the release function is called which resets `private_data` to `NULL`.

```C
static int goldfish_pipe_open(struct inode *inode, struct file *file)
{
    /* Allocate new pipe kernel object */
    struct goldfish_pipe *pipe = kzalloc_obj(*pipe);
...
  file->private_data = pipe;
...
}

static int goldfish_pipe_release(struct inode *inode, struct file *filp)
{
...
  file->private_data = NULL;
...
}
```

When callback functions like read, write, poll are called, the device specific
struct is retrieved again to be used in the function.
```C
static ssize_t goldfish_pipe_read_write(struct file *filp,
    char __user *buffer,
    size_t bufflen,
    int is_write)
{
struct goldfish_pipe *pipe = filp->private_data;
    ...
}
```

Note: While this example uses `file->private_data`, the pattern is equivalent
to `void* user_data` in callback registration. Both pieces of code serve to
__pass driver-specific state__ to the handler.

You might wonder now, how does Rust handle this use case in a explicit and type-safe way?

# Rustification of `user_data`

Rust is based on the principles of __ownership__ and __thread safety__. If we want a state to be
passed to a C function, we must make sure it is in the __heap__ and __cannot be moved around__ by
the Rust compiler. This is why the driver specific data is allocated in a `Box` and __pinned__.
The `Box` reserves memory on the heap for our user data. The pinning makes
sure, that the data in the box cannot be moved out. For the C code, the user
data is a __black box__. When the Rust driver probes a new device and calls
`set_drvdata`, it effectively hands a pointer to "some data" to the C code, which does not
have any meaning to it. At the same time, the ownership is also transferred to the
C code.

When the C code on occurrence of an interrupt calls the interrupt handler function of
our driver, it __hands back__ this opaque data to us. Since we know how to interpret the data,
this is where our otherwise generic function gets the context from. If the device is unplugged
or disappears, the remove function is called, which also gets passed this black box. It is
then the Rust drivers task to free the memory.

Let's look at `rust/kernel/platform.rs`.

```rust
impl<T: Driver + 'static> Adapter<T> {
    extern "C" fn probe_callback(pdev: *mut bindings::platform_device) -> kernel::ffi::c_int {
        // SAFETY: The platform bus only ever calls the probe callback with a valid pointer to a
        // `struct platform_device`.
        //
        // INVARIANT: `pdev` is valid for the duration of `probe_callback()`.
        let pdev = unsafe { &*pdev.cast::<Device<device::CoreInternal>>() };
        let info = <Self as driver::Adapter>::id_info(pdev.as_ref());

        from_result(|| {
            let data = T::probe(pdev, info);

            pdev.as_ref().set_drvdata(data)?;
            Ok(0)
        })
    }
...
}
```

The data specific to the specific driver instance is set in `pdev.as_ref().set_drvdata(data)?;`.

The implementation of `set_drvdata` can be found in `rust/kernel/device.rs`.
The function `into_foreign()` is part of the `ForeignOwnable` trait. It
__relinquishes ownership__ of data to the device, which in turn gives the
__ownership to the C code__. It ensures that the Rust object is "pinned" in memory
and __won't be dropped__ by Rust's borrow checker while the C kernel is handling
it.

The code `data.into_foreign()` converts the data from a Rust Box into a raw
pointer, a.k.a `*const c_void` or a typed pointer `*const T`. The C kernel
API, which is called by using bindings, almost always expects a `*mut
core::ffi::c_void` to be passed as a function argument.

Let's assume for example that `into_foreign()` returned a typed pointer `*const
MyDriverData`, while the C function expects a `mut void *`. Rust is much stricter
than C and will therefore not implicitly convert a typed pointer into a generic
pointer. Furthermore, `into_foreign()` often returns a __read-only pointer__ to ensure safety,
while many C registration functions expect a `*mut void`. 
The program must call `cast()` explicitly to convert it a generic mutable
pointer.


```rust
impl Device<CoreInternal> {
...
    /// Store a pointer to the bound driver's private data.
    pub fn set_drvdata<T: 'static>(&self, data: impl PinInit<T, Error>) -> Result {
        let data = KBox::pin_init(data, GFP_KERNEL)?;

        // SAFETY: By the type invariants, `self.as_raw()` is a valid pointer to a `struct device`.
        unsafe { bindings::dev_set_drvdata(self.as_raw(), data.into_foreign().cast()) };
        self.set_type_id::<T>();

        Ok(())
    }
...
}
```

```rust
impl Device<CoreInternal> {
...
    /// Borrow the driver's private data bound to this [`Device`].
    ///
    /// # Safety
    ///
    /// - Must only be called after a preceding call to [`Device::set_drvdata`] and before the
    ///   device is fully unbound.
    /// - The type `T` must match the type of the `ForeignOwnable` previously stored by
    ///   [`Device::set_drvdata`].
    pub unsafe fn drvdata_borrow<T: 'static>(&self) -> Pin<&T> {
        // SAFETY: `drvdata_unchecked()` has the exact same safety requirements as the ones
        // required by this method.
        unsafe { self.drvdata_unchecked() }
    }
}

impl Device<Bound> {
    /// Borrow the driver's private data bound to this [`Device`].
    ///
    /// # Safety
    ///
    /// - Must only be called after a preceding call to [`Device::set_drvdata`] and before
    ///   the device is fully unbound.
    /// - The type `T` must match the type of the `ForeignOwnable` previously stored by
    ///   [`Device::set_drvdata`].
    unsafe fn drvdata_unchecked<T: 'static>(&self) -> Pin<&T> {
        // SAFETY: By the type invariants, `self.as_raw()` is a valid pointer to a `struct device`.
        let ptr = unsafe { bindings::dev_get_drvdata(self.as_raw()) };

        // SAFETY:
        // - By the safety requirements of this function, `ptr` comes from a previous call to
        //   `into_foreign()`.
        // - `dev_get_drvdata()` guarantees to return the same pointer given to `dev_set_drvdata()`
        //   in `into_foreign()`.
        unsafe { Pin::<KBox<T>>::borrow(ptr.cast()) }
    }
...
}
```

An example of a platform driver that uses `platform.rs` is to be found in
`drivers/pwm/pwm_th1520.rs`, a T-HEAD TH1520 PWM driver. I guess it is no
surprise that the first user of the Rust based platform driver is a [RISC-V SoC
PWM controller](https://riscv.org/blog/the-release-of-the-first-two-mass-produced-development-boards-aosp-powered-by-th1520-soc/).

The driver's private data is defined as `Th1520PwmDriverData`.

```rust
#[pin_data(PinnedDrop)]
struct Th1520PwmDriverData {
    #[pin]
    iomem: devres::Devres<IoMem<TH1520_PWM_REG_SIZE>>,
    clk: Clk,
}
```

When the driver's `probe()` function is called, it passes `Th1520PwmDriverData`
to `pwm::Chip::new()`. Internally, this stores the data via `pwmchip_alloc()`
and constructs it in-place in the private data area.

```rust
// drivers/pwm/pwm_th1520.rs, line 368-375
let chip = pwm::Chip::new(
    dev,
    TH1520_MAX_PWM_NUM,
    try_pin_init!(Th1520PwmDriverData {
        iomem <- request.iomap_sized::<TH1520_PWM_REG_SIZE>(),
        clk <- clk,
    }),
)?;
```

This is the Rust equivalent of C's `void* user_data` - the platform adapter
stores the driver-specific state and passes it back via `drvdata()` when
the callback is invoked.

```rust
impl pwm::PwmOps for Th1520PwmDriverData {
    fn round_waveform_tohw(...) {
        let data = chip.drvdata();  // retrieves Th1520PwmDriverData
        let rate_hz = data.clk.rate().as_hz() as u64;
        // ... use rate to convert ns to cycles
    }
}
```

As a small side note, I want to mention that the Rust compiler and runtime can
only check that the rules and boundaries are enforced in the __Rust environment__.
If the FFI is used to call C functions in an `unsafe` block, the C code can
manipulate the data in memory. The functions `into_foreign()` and
`from_foreign()`, defined in the trait `ForeignOwnable`, do provide safe
bridges between both programming languages. But by using an `unsafe` block, you
tell the compiler that __you will take the responsibility__ for the safety of the
code.

This is also why the implementation is marked as `unsafe impl`. This mark is placed
when the __compiler cannot verify__ that the programmer is following the rules of the
trait remarked in the comments.

Furthermore, the file `rust/kernel/types.rs` contains a default implementation
of `ForeignOwnable` for a __stateless driver__. If the driver passes `()` as user
data, this implementation makes sure that Rust knows how to turn it back into
'nothing' gracefully.

The documentation in `rust/kernel/types.rs` explicitly states, that the only guarantees
given about the pointer are, that it has a __minimum alignment__, and that the return pointer
is __not null__. The pointer could still be __invalid, dangling or pointing to uninitialized
memory__, which the caller of `from_foreign` needs to handle. Furthermore, the programmer
must ensure that the pointer provided to `from_foreign` must have been returned
from a previous call to `into_foreign`, and that the pointer has not been passed
to `from_foreign` more than once. If the programmer does not uphold this safety contract,
we have __undefined behavior__ of the code, as in good old C programs.

The last safety mechanism I want to mention is implemented by the line `unsafe
impl<T: 'static, A> ForeignOwnable for Box<T, A> { .. }` in `kbox.rs`. In Rust,
`'static` denotes an "eternal" lifetime. This is the longest possible lifetime
a value can have, lasting for the __entire duration__ of the program's
execution. Because of this implementation of ForeignOwnable for Box, the type
that is stored in the box needs to have a static lifetime. This prevents using
types with borrowed references that cannot safely cross the FFI boundary, which
is checked at compile time. In other words, if a type or any of its members
contain a non-`static` lifetime reference at compile time, it would be unsafe
to pass to C code since the __lifetime cannot outlive__ the foreign context.

## Summary

In C, `void* user_data` passes context to callbacks but __loses type information__.
Rust replaces this with `set_drvdata()` + `drvdata<T>()`, which adds __type-safe
checks__ at compile time. The `ForeignOwnable` trait bridges the worlds between
Rust and C language safely.

---

[^1]: A physical controller might have a specific IRQ number/interrupt line
    assigned to it, a memory-mapped I/O (MMIO) base address where the device's
registers can be read from and written to, or references to the power
management or clock.

[^2]: Hardware is inherently stateful. Therefore, a driver might need to keep
    track of the state the device currently is in (INITIALIZING, READY,
SUSPENDED, ERROR, ..). Furthermore, mutexes or spinlocks are used to make sure
that only one part of the kernel talks to a device at once.

[^3]: Usually in the kernel, a device driver registers with a parent driver or subsystem, and keeps a back pointer to it. A GPIO driver will have a refernce to the GPIO subsystem, a network driver will reference a `struct net_device`, etc.

[^4]: In case of devices using DMA mapped memory, a pointer to the memory shared between the CPU and the hardware might be contained in user data. If wait queues are used, the process might be "sleeping" until the specific hardware finishes its task.


