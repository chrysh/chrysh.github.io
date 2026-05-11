---
title: The Cleanup Pattern - From goto out to Drop
date: '2026-05-11'
categories:
  - Rust
  - C
  - Userspace
  - Memory Management
tags:
  - Rust
  - C
  - Memory Management
  - Migration
---

[<img class="penguin" style="float: left; padding-left: 0%; width: 30%" src="/static/img/rusty_penguin_17.jpeg" alt="Rusty penguin. Created by DALL·E 3."/>](https://github.com/Rust-for-Linux/)


# The C Cleanup Pattern

While in most programming styles using goto's is frowned upon, in the Linux
Kernel this is the default method of undoing the registration and creation work
done by each driver. You oftentimes have the situation that you start
registering a driver. When an error occurs in this process, you have to roll
back everything you just did in reverse order. This is where `goto` comes in handy.
By stacking the `goto` labels, you only undo the work you __have already done__.

An example of the cleanup pattern used in the Linux Kernel C code can be found
in `drivers/block/aoe/aoeblk.c`, which implements block device routines for
the ATA over Ethernet (AoE) network protocol. This driver allows a computer to
access hard drives over a standard Ethernet network as if they were locally attached
IDE/SATA drives.

```C
/* blk_mq_alloc_disk and add_disk can sleep */
void
aoeblk_gdalloc(void *vp)
{
    // ...
    mp = mempool_create(MIN_BUFS, mempool_alloc_slab, mempool_free_slab,
        buf_pool_cache);
    if (mp == NULL) {
        printk(KERN_ERR "aoe: cannot allocate bufpool for %ld.%d\n",
            d->aoemajor, d->aoeminor);
        goto err;
    }

    // ...
    err = blk_mq_alloc_tag_set(set);
    if (err) {
        pr_err("aoe: cannot allocate tag set for %ld.%d\n",
            d->aoemajor, d->aoeminor);
        goto err_mempool;
    }

    gd = blk_mq_alloc_disk(set, &lim, d);
    if (IS_ERR(gd)) {
        pr_err("aoe: cannot allocate block queue for %ld.%d\n",
            d->aoemajor, d->aoeminor);
        goto err_tagset;
    }

// ...
    err = device_add_disk(NULL, gd, aoe_attr_groups);
    if (err)
        goto out_disk_cleanup;
    aoedisk_add_debugfs(d);
// ...
    return;

out_disk_cleanup:
    put_disk(gd);
err_tagset:
    blk_mq_free_tag_set(set);
err_mempool:
    mempool_destroy(mp);
err:
    spin_lock_irqsave(&d->lock, flags);
    d->flags &= ~DEVFL_GD_NOW;
    queue_work(aoe_wq, &d->work);
    spin_unlock_irqrestore(&d->lock, flags);
}
```

We see stacked cleanup code. Instead of having a pyramid of deeply nested
`if` statements, the success path is mostly flat. Only in the case of
errors, we jump to the __cleanup label stack__. If creating the memory pool failed,
we have to queue the work again for a next try. If the memory pool was created
successfully, but we fail to allocate a tag set, we have to __roll
back__ the successfully executed code paths in reverse order. You can clearly see
where a resource is acquired and what cleanup is necessary to reverse this step.

And the functions called are usually also very similar to what has to be executed
when you want to unregister the device or unload a module. Wouldn't it be great to
have this cleanup routine gathered in one place?

# Enter Rust's Drop Trait

Rust introduces actually useful object-oriented patterns to low level programming.
It follows the principles of the __RAII (Resource Acquisition is
Initialization)__ pattern where the lifecycle of a resource (memory, file
handles, locks) is tied to the lifecycle of a value.

When a value is no longer needed (e.g. goes out of scope), Rust will run a
"destructor" on it. The destructor consists of two components. First, if the
`Drop` trait is implemented for its type, the function `Drop::drop` is called
for that value. After that, the destructors of all fields of that value are
called in order of their declaration in the struct. The Rust compiler generates
the "drop glue" code for this type automatically. In practice, `Drop::drop` only needs
to be implemented for types that directly manage resources[^1]. If your driver
allocated memory, opened a file or a network socket, the cleanup of those
resources should happen here.


```rust
#[lang = "drop"]
#[stable(feature = "rust1", since = "1.0.0")]
#[rustc_const_unstable(feature = "const_destruct", issue = "133214")]
pub const trait Drop {
    #[stable(feature = "rust1", since = "1.0.0")]
    fn drop(&mut self);
}
```

We can find several examples of the usage of this trait in the current upstream Linux
kernel 7.0-rc6. For now, most current Rust kernel drivers merely call the release function
or free up resources using C bindings.

The `Drop` trait is implemented for the clock driver in `rust/kernel/clk.rs`. All this driver
has to do is decrement the reference count for the clock instance, which is done
in this case by calling the C function `clk_put` using `bindings`.

```rust
impl Drop for Clk {
    fn drop(&mut self) {
        // SAFETY: By the type invariants, self.as_raw() is a valid argument for [`clk_put`].
        unsafe { bindings::clk_put(self.as_raw()) };
    }
}
```

Compared to the clock driver, the implementation of `drop` for the red black tree
has to do more work. It has to iterate through all nodes in the tree, remembering the
next node and freeing the memory allocated on the heap for each node by calling the drop
function on each KBox element.

```rust
impl<K, V> Drop for RBTree<K, V> {
    fn drop(&mut self) {
        // SAFETY: `root` is valid as it's embedded in `self` and we have a valid `self`.
        let mut next = unsafe { bindings::rb_first_postorder(&self.root) };

        // INVARIANT: The loop invariant is that all tree nodes from `next` in postorder are valid.
        while !next.is_null() {
            // SAFETY: All links fields we create are in a `Node<K, V>`.
            let this = unsafe { container_of!(next, Node<K, V>, links) };

            // Find out what the next node is before disposing of the current one.
            // SAFETY: `next` and all nodes in postorder are still valid.
            next = unsafe { bindings::rb_next_postorder(next) };

            // INVARIANT: This is the destructor, so we break the type invariant during clean-up,
            // but it is not observable. The loop invariant is still maintained.

            // SAFETY: `this` is valid per the loop invariant.
            unsafe { drop(KBox::from_raw(this)) };
        }
    }
}
```

# How Drop Works

Dropping a value can be triggered in different scenarios. One trigger is when a
value goes __out of scope__, which happens at the __end of a block__ `{..}` or
when a __function returns__. Furthermore, when you assign a new value to an
__existing variable__, the old variable is dropped. It is also possible to call
`core::mem::drop(x)` in the kernel explicitly. In most cases, this is not
advised, because it circumvents the compiler's drop check.

One of the most powerful aspects of this system is how it interacts with the
__question mark operator (`?`)__. In C, if you have five initialization steps and
the fourth fails, you must manually `goto` the specific label that undoes
exactly three steps. 

In Rust, if you return early with `?`, the compiler's __drop check__ knows
exactly which local variables have been initialized at that specific line of
code. It will call `drop` on only those __active variables__ in __reverse
order__. Conversely, if the function succeeds and returns a value, that value
is __moved to the caller__. Rust recognizes this __transfer of ownership__ and
ensures the destructor is __not called__ for the returned value, while still
cleaning up all other local temporaries in the scope.

The rust lang file `core/ops/drop.rs` gives an idea of the order of destruction.
Local variables are dropped in __reverse order of declaration__, like on a stack,
ensuring that if a later variable depends on an earlier one, the dependency is
still valid during the later variable's cleanup.

Struct fields and collections on the other hand are dropped in __order of declaration__.
The cleanup process for structs is a systematic, recursive traversal
of the value's fields to ensure that every resource it owns is properly released.
This specialized destructor that recursively visits every part of the structure
is called __drop glue__ and is generated at compile time.

The "drop check" analysis determines whether it is safe to drop a value and its
fields recursively. In particular, the compiler needs to keep track of __types and lifetimes__,
and which of them still needs to be live when `T` gets dropped. If a value does not
need to free any resources, there is no need to add a Drop implementation.

The strategy in Rust for making sure no value is forgotten to be dropped
is to traverse structs recursively until it reaches a __leaf node__. Leaf nodes are
types that do not own anything, and therefore don't require their own `drop` logic.
Examples of leaf nodes are __primitive types__ like `i32`, `bool`, __references and raw pointers__
and __function items and function pointers__, which are just addresses of code in memory.

If a type implements the `Drop` trait, its `Drop::drop` method is called __before__
its fields `drop` methods, making sure that the fields can be used one last time
during the cleanup process. Think of a struct owning a file descriptor.
You want to close the file before its memory is freed and the descriptor becomes invalid.


The drop behavior of enums depends on which variant. In the case of `Result<T, E>` or
`Option<T>`, each of `Some(T)` vs `None` or `Ok(T)` vs `Err(E)` requires a different
drop logic. Only active variants are dropped, and unused variants don't need a cleanup.
[^2]

Rust, being the "having its cake and eating it too" language that it is, ensures
at compile time that the `drop` function will be called when our value
goes out of scope, removing the human error of forgetting to free memory or
flushing a buffer.


# Beyond Memory: All Resources

Freeing memory or releasing a resource is not the only action the programmer
might want to do. Different drivers will have different requirements. For
example, if a bus driver is released, it will want to call the release
functions of all the devices attached to it. When a network socket goes out of
scope, it will want to gracefully shut down the TCP connection by initiating a
TCP FIN handshake, flush the receive and transmit buffers and release the
port/IP tuple.

If a file handle goes out of scope, the reference count on the underlying
`struct file` is decremented. If this count reaches zero, which means all the
users of the file are gone, the kernel calls the `.release` method, flushes
pending writes, releases all locks and frees the memory associated with the handle.

For custom hardware resources, virtual memory mappings to physical hardware registers
might need to be unmapped. If there was a DMA channel used with the hardware, the
active transfers need to be stopped and the DMA engine channel released, so that other
drivers can use it.

Besides resources, Rust uses so-called __guards__ to protect a resource or a state change.
For example, if a programmer acquires a mutex, they receive a `MutexGuard`. You __never
manually__ call `unlock` on it. Instead, this lock is __tied to the lifetime__ of the
guard object. When the guard object goes out of scope, Rust automatically calls the
mutex's `Drop` implementation to release the lock, just as `mutex_unlock` would do.
This makes it impossible to forget to unlock, even in complex error paths. However
the function is exited, be it a successful return, an early `?` error, or a panic, the
lock is __automatically released__ when the guard's lifetime ends.

# Interfacing with C: ForeignOwnable Pattern

We had a dive into
[`ForeignOwnable`](../../../../2026/04/29/the-callback-conundrum-porting-void-pointers-to-user-data/)
previously, where we learned that this trait is used to transfer ownership
between Rust and C code. This functionality makes sure that the Rust value is pinned
in memory, which means from the time of creation, it will always have the same memory address,
and that it won't be dropped by Rust's borrow checker while the C part of the kernel
is handling it, even though to the Rust part it looks like the variable is unused.
This trait builds a safe bridge between both programming languages.

# Comparison: C vs Rust Cleanup

In C, when you add a __new resource__, you have to __edit every cleanup stack__ in the
call chain. The cleanup code tends to be repetitive if it occurs in several
places, and it is harder to keep all occurrences in sync when resources change.
It is way too easy to miss a cleanup label, or reverse the order of the
cleanup, and cause use-after-free or double-free bugs.
Rust automates all those scenarios, removing entire bug classes. Error handling becomes
simpler, because early returns using the `?` operator, `match` arms and panics all
trigger the __same deterministic teardown code__. Mutexes and Spinlocks get unlocked
in the `Drop` implementation, preventing deadlocks. And since the drop glue code is
generated at compile time and inlined, we don't have a runtime penalty for these
safety guarantees.

# TL;DR

While C makes it __easy to forget__ some of the cleanup when a value goes out of scope,
where the responsibility lies on the shoulders of the programmer, Rust makes it
very hard to forget the cleanup. In Rust, the programmer only has to __remember once__ to
implement the cleanup code, and the Rust equivalent of a compile time garbage collector
makes sure this code is executed any time the value is not needed anymore. Fire and forget
the type safe way.

---

[^1]: You cannot implement `Drop` and `Copy` for the same type. `Copy` implies duplication.
If a type is `Copy`, the compiler assumes it can safely copy the value bit by bit. Both
the original and the copy are considered valid and independent. `Drop` implies resource
management. It is used when a type owns a resource that requires manual cleanup, like heap
memory, file handles, or network sockets. If a type managed a resource (implemented `Drop`) and
was also `Copy`, Rust would call `drop` on both when they go out of scope, attempting to free
the same resource twice. When `Move` is used, the ownership of the resources is transferred
and a call to `drop` is therefore allowed.

[^2]:For an in-depth explanation of the drop check, I recommend reading the
[Rustonomicon](https://doc.rust-lang.org/nomicon/dropck.html), the "advanced"
Rust book focusing on Unsafe Rust.
