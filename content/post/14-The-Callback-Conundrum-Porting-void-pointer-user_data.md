---
title: The Callback Conundrum - Porting void pointers to user data
date: '2026-04-30'
categories:
  - Rust
  - C
tags:
  - Rust
  - C
---

[<img class="penguin" style="float: right; padding-left: 5%" src="/static/img/rusty_penguin_15.jpeg" alt="Rusty penguin. Created by DALL·E 3."/>](https://github.com/Rust-for-Linux/)

Platform drivers were one of the first success stories of the Rust-for-Linux project.
They were a proof of concept that the integration of Rust into the existing Linux kernel
C code base would work.
The decision to focus on platform drivers early on was strategic. They are simpler compared to complex GPU or network drivers, and therefore provide a perfect sandbox for testing Rust abstractions.

Therefore, we want to show the callback conundrum based on the Rust implementation of
the platform driver base `rust/kernel/platform.rs` in Linux kernel Version 7.0.

# The function of `void *user_data`
Most C callback functions have the following signature:
`void register_callback(void (*cb)(int, void*), void* user_data);`. This
signature is a common C pattern to pass a private state or context to a stateless
function pointer. You can imagine the `user_data` like a bucket of bytes, that
has no information for the calling function of the callback in C. The usage of
`user_data` is a way of handling the lack of native polymorphism and
inheritance in C.

The userdata is usually cast to a driver dependent struct, depending on which
driver callback was callend, and contains the __private state__ of a specific
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
is initialized with a pointer to th goldfish specific `struct goldfish_pipe`. When the
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
You might wonder now, how does Rust handle this use case in a explicit and type-safe way?

# Rustification of `user_data`

Rust is based on the principles of ownership and thread safety. If we want a state to be
passed to a C function, we must make sure it is in the heap and cannot be moved around by
the Rust compiler. This is why the driver specific data is allocated in a `Box` and pinned.
The `Box` reserves memory on the heap for our user data. The pinning makes
sure, that the data in the box cannot be moved out. For the C code, the user
data is a black box. When the Rust driver probes a new device, it calls
`set_drvdata`, it effectively hands a pointer to "some data" to the C code, which does not
have any meaning to it. At the same time, the ownership is also transferred to the
C code.

When the C code on occurence of an interrupt calls the interrupt handler function of
our driver, it hands back this opaque data to us. Since we know how to interpret the data,
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
ownership to the C code. It ensures that the Rust object is "pinned" in memory
and won't be dropped by Rust's borrow checker while the C kernel is handling
it.

The code `data.into_foreign()` converts the data from a Rust Box into a raw
pointer, a.k.ka `*const c_void` or a typed pointer `*const T`. The C kernel
API, which is called by using bindings, almost always expects a `*mut
core::ffi::c_void` to be passed as a function argument.

Let's assume for example that `into_foreign()` returned a typed pointer `*const
MyDriverData`, while the C function expects a `mut void *`. Rust is much stricter
than C and will therefore not implicitly convert a typed pointer into a generic
pointer.
Furthermore, `into_foreign()` often returns a read-only pointer to ensure safety,while many C registration functions expect a `*mut void`. 
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

<!---
Key patterns for callbacks:
2. Line 95-109: extern "C" fn probe_callback - receives raw pointer
3. Line 106: set_drvdata - stores driver data (replaces C's void* user
   data)
4. Line 121: drvdata_borrow - retrieves it (type-safe)


This is exactly the callback pattern for your blog. The C way uses void*
user_data, the Rust       way uses set_drvdata/drvdata_borrow with generic T.
-->

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
