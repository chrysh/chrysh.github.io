---
title: C to Rust - Memory
date: '2025-01-28'
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
  - Coreutils
---

[<img class="penguin" style="float: left; padding-left: 0%" src="/static/img/rusty_penguin_13.jpeg" alt="Rusty penguin. Created by DALL·E 3."/>](https://github.com/Rust-for-Linux/)

One of the big worries of any seasoned embedded developper is always memory and processing overhead. This blog post will show how you can have the cake and eat it as well: You can have the safety guarantees thanks to the Rust compiler without suffering any memory or CPU overhead. The Rust compiler takes care of making sure that you don't read or write outside of a buffer or try to reference a NULL pointer.

# Zero cost abstractions
In C, `p == NULL` checks for a pointer that points to address 0x0. In Rust, `Option<Box<T>>` actually uses an optimization where `None` is represented as a null pointer at the machine level.

This means there is zero memory overhead for using Option in Rust compared to a nullable pointer in C.
```
sizeof(Option<Box<T>>)=sizeof(void*)
```

In C, a pointer is just a memory address. You use NULL (usually 0) to represent the absence of a value.
```
if (p == NULL) {
    // Handle error
}
```

# Memory management

| Feature         | C Language                      | Rust Language                         |
| -------         | ----------                      | -------------                         |
| Heap Allocation | `int *p = malloc(sizeof(int));` | `let p = Box::new(0i32);`             |
| Deallocation    | `free(p);`                      | Automatic (when `p` leaves the scope) |
| Memory          | `if (p == NULL) { ... }`        | `Option<Box<i3>` (Pattern matching)   |


At the machine level, the variable p is exactly 8 bytes (on a 64-bit system) or 4 bytes (on 32-bit).
In Rust, we don't use null pointers because they cause crashes. Instead, we use Option<Box<T>>.
Typically, an Enum in Rust requires a "discriminant" (a small integer) to tell the program which variant is being used (`Some` or `None`). You would expect the memory layout to look like this:

    Tag: 1 byte (to say if it's Some or None)
    Padding: 7 bytes (for alignment)
    Pointer: 8 bytes Total: 16 bytes (Double the size of C!)

However, the Rust compiler knows that a Box can never be null. Therefore, it steals the null value (0x0) and uses it to represent `None`.

| Type                    | Memory Layout (64-bit) | Value of "None" / "Null" |
| -------                 | ----------             | -------------            |
| C Pointer (int*)        | 8 bytes                | 0x00000000               |
| Rust (Option<Box<i32>>) | 8 bytes                | 0x00000000               |

The Rust compiler makes sure the code is efficient in space and performance.
As shown above, the `Option<Box<T>>` occupies the exact same number of bits as a C pointer. There is __zero memory overhead__.

When you "match" on the Option, the assembly code generated is often a simple "jump if zero" instruction — exactly what the check `if (p == NULL)` in C compiles down to. There is __zero CPU overhead__.

While the resulting machine code may be identical, the developer experience is fundamentally different. C relies on a __Social Contract__, where the developer is trusted to check for `NULL` before dereferencing; failing to do so triggers undefined behavior or security flaws. Rust upgrades this to an __Enforced Contract__. The compiler forces you to handle the `None` case before you can even access the data, ensuring the code is safe before it ever builds.

# Pointers vs. References

The core of "Rustification" is moving from dangerous raw pointers to tracked references.

Immutable is like a read only access to a value, while the mutable reference is like a read-write mutex on the object.

| Language        | Code        | Notes                                    |
| -------         | ----------  | -------------                            |
| Raw Pointer (C) | `int* ptr;` | Can be NULL, uninitialized, or dangling. |
| References (Rust) | &T (immutable) or &mut T (exclusive mutable)         | Guaranteed to be valid and non-null by the compiler. |

In C, const `int*` says "I promise not to change this," but someone else might still have a non-const pointer to it and change it behind your back.

In Rust, an immutable reference `&T` is truly immutably shared. Unless you use "Interior Mutability" (like Cell or Atomic), you have a guarantee that the value will not change as long as that reference exists. This is even stronger than C's const.


| Feature     | Read-Only Reference (&T)     | Mutable Reference (&mut T)    |
| ----------  | ----------                   | ----------                    |
| Analogy     | Shared Lock (Read)           | Exclusive Lock (Write)        |
| Quantity    | Unlimited                    | Exactly one                   |
| Concurrency | Many can read simultaneously | No one else can read or write |
| Access      | Read-only                    | Read-Write                    |

The "Static" Benefit: You should emphasize that while a Mutex or RWLock has a runtime cost (checking the lock state, blocking threads), Rust's "lock" is erased at compile-time.

The Rule of Aliasing: In C, the compiler often can't optimize code because it doesn't know if two pointers alias (point to the same memory). In Rust, the "Exclusive Lock" rule (&mut) guarantees to the compiler that no other pointer aliases that memory, allowing for much more aggressive optimization.

In fact, the Rust compiler effectively acts as a static borrow checker that enforces the same rules at compile-time that a `pthread_rwlock_t` enforces at runtime.

Technically speaking, Rust's borrow checker is a zero-cost, compile-time Read-Write Mutex. It provides the safety of a lock without the latency of a syscall.

# Arc/ Rc

Arc stands for "Atomic Reference Counted", Rc stands for "Reference Rounted".
On creation, when you call `Arc::new(data)`, the strong count starts at 1.
When you call `.clone()` on the data later, you aren't copying the data; you are simply incrementing the strong count using an atomic operation.
When an Arc handle goes out of scope, it decrements the strong count.
When the strong count reaches 0 is the data actually "dropped" and the memory freed.

The "A" in Arc stands for Atomic. This is the crucial difference between Arc and its single-threaded sibling, Rc.
Rc (Reference Counted) uses standard integers for counters. It's fast, but if two threads try to increment the counter at the same time, you get a data race (and likely a memory leak or double-free).
Arc (Atomic Reference Counted) on the other hand uses CPU-level atomic instructions (like LOCK INC on x86 or LDREX/STREX on ARM). This ensures that the reference count remains accurate even if multiple threads are cloning or dropping handles simultaneously.

# Mutable Arc

A common point of confusion for C developers is that Arc<T> only grants immutable access to the data it holds.

Remember this analogy: "Immutable is like a read-only access." Because multiple threads have an Arc handle to the same data, Rust cannot allow any of them to have a mutable reference (&mut T), as that would cause a data race.

So how to get Mutability? To change the data inside an Arc, you must combine it with a type that provides Interior Mutability. This is the classic Rust "Lego" pattern:
```
    Arc<Mutex<T>>
```

The Arc gets the data to the threads safely, and the mutex ensures only one thread can write to the data at a time.

Using Arcs does not come for free. Arc requires a  heap allocation for the control block (unless you are using a specialized no_std crate for static Arcs). Atomic increments/decrements are more expensive than normal additions because they require cache synchronization across CPU cores.

When you call `let shared = Arc::clone(&original);`, the CPU must increment the reference count safely across cores.
On x86, this uses the lock prefix to ensure the increment is atomic across the memory bus:

```
lock inc qword ptr [rax]  ; Atomically increment the 64-bit count at address in RAX
```

Embedded Linux often runs on ARM (like a Raspberry Pi or an i.MX8). The ARM instruction set uses a "Load-Link / Store-Conditional" or a specific atomic add instruction:

```
prfm  pstl1strm, [x0]     ; Preload for write
ldxr  x8, [x0]            ; Load exclusive (the current count)
add   x8, x8, #1          ; Increment the register
stxr  w9, x8, [x0]        ; Store exclusive (fails if memory changed)
cbnz  w9, retry_label     ; If it failed (w9 != 0), try again
```

Dropping is more complex because the CPU must check if the count reached zero. If it is zero, it must jump to the destructor (to free the memory).

In x86_64 assembly, this is how it looks like:

```
lock xadd [rax], -1       ; Atomically decrement and get previous value
cmp rax, 1                ; Was the previous value 1? (Meaning it's now 0)
jne skip_free             ; If not zero, we are done
call destructor_function  ; If zero, clean up the heap
```

We ommit the ARM assembly code example here, because it is too long to explain.

A small excursion into weak pointers. In your project, you might get into the circular reference problem, where object A has an Arc to B, and B has an Arc to A. Since both objects have somebody pointing at them, their reference count would always be at least 1.

Using Weak pointers resolves this problem. If B uses a weak handle, which does not increase the strong reference count, the cycle can be broken and the memory can be freed.

# Strings and Buffers and Zero-Copy via Slicing (`&[u8]`)

Rust separates fixed-size strings from growable ones, preventing the classic buffer overflow.
Fixed String: 
```
const char* s = "Hello"; → let s: &str = "Hello";
```

Growable String:
```
char* s = strdup("Hello"); → let s: String = String::from("Hello");
```

In C, passing a subset of a buffer often involves passing a pointer and a length separately `(char∗ and size_t)`. It’s error-prone.
When accessing `buffer[i]` in Rust, it performs automatic bounds checking that panics instead of corrupting memory. 
Rust uses "Fat Pointers" (16 Bytes on 64-bit), and each slice carries its own bounds.
This is perfect for network stacks or filesystem drivers where you are "peeling off" headers without copying data.


# Stack vs. Heap (The Copy vs. Move Semantics)

C developers love the stack because it’s fast. Rust is even more aggressive about stack allocation than C++ often is.

In C, when you pass a struct to a function by value, it's a memcpy. In Rust, it’s a "Move."
"Moving" (a.k.a transfering ownership of) a variable in Rust allows the compiler to optimize away the copy entirely. If you move a large struct into a function, and the original is never used again, the compiler often just lets the function "inherit" the memory location.

# No Garbage Collection

Rust does not have a garbage collector (GC), it uses the "Resource Acquisition is Initialization" (RAII) approach. 
Unlike languages like Java, Python, or Go, which use a background process to scan memory and reclaim unused space while the program is running, Rust manages memory through a system of Ownership and Borrowing.
Memory is automatically deallocated the exact moment the variable owning it goes out of scope. This is handled entirely at compile time.
For situations, where ownershoip is not straightforward, Rust provides smart pointers `Rc<T>` for single threaded and `Arc<T>` for multithreaded scenarios.

# Custom Allocators and no_std

In Embedded Linux, you aren't always in "User Space" with a standard malloc.
Sometimes you need to allocate from a specific pool (e.g., DMA-capable memory or a fixed static buffer). In Rust, you can tell the entire program to use a specific memory pool by implementing the GlobalAlloc trait. This is a nightmare to enforce globally in a large C project.


# Memory Layout Control (repr(C))

To match hardware registers, embedded developers have to ensure that the padding and alignment of the structure conforms to the physical reality. 
By default, Rust reorders struct fields to minimize padding and to save RAM, as is done in C.
For hardware-level work, you must use `#[repr(packed)]` where in a C program you would have written `__attribute__((packed))` in order to remove all padding and minimize the struct size. Furthermore, `#[repr(align(n))]` forces the struct to start at a specific memory boundary, as the `alignas(n)` specifier would do in a C++ project.
