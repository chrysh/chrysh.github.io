---
title: Porting Bitfields and Register Maps
date: '2026-05-01'
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

[<img class="penguin" style="float: left; padding-left: 0%" src="/static/img/rusty_penguin_15.jpeg" alt="Rusty penguin. Created by DALL·E 3."/>](https://github.com/Rust-for-Linux/)

Because of the nature of hardware registers, embedded C codebases are saturated with bitfields and `#define` masks. Rust lacks a direct equivalent in the core language.

> TBD

# Mapping C struct bitflags

The Linux kernel recently added the `bitfields!` and `register!` macros. The first and primary "real world" user is the Nove driver, which is the Rust based successor to Noveau for NVIDIA GPUs.

GPUs have thousands of complex registers. Having a specific macro for manipulating registers with "getters" and "setters" prevents developers from bugs like writing a 32-bit value to a 16-bit register, or toogling a a "reserved" bit.

Furthermore, the Rust PCI sample driver also recently was updated in order to use `register!`.

> TBD: Check out PCI driver


The usage is as follows:

Based on the current implementation of bitfields in `drivers/gpu/nova-core/bitfields.rx`, which 
implements the bitfield library for Rust structures in the Linux kernel, you would use
the control register as follows:

```
bitfield! {
    pub struct ControlReg(u32) {
        7:7 state as bool => State;
        3:0 mode as u8 ?=> Mode;
    }
}
```




```
bitfield! {
    pub struct Status(u32) {
        /// The 'enabled' flag at bit 0
        enabled: bool [0..1],
        /// The 'mode' value in bits 2 through 4
        mode: u8 [2..5],
    }
}
```
The 

```
// Modern 2026 Kernel Syntax
register! {
    pub CTRL(u32) @ 0x04 {
        enabled: bool [0..1],
        mode: u8 [2..5],
    }
}
```

```
dev.regs.status().modify(|r| r.set_mode(Mode::Active));
```



# Replacing shift macros

# Using volatile to prevent compiler optimization

# svd2rust



#### TBD


While in C operations on bitfield are managed by the compiler, in Rust, the
programmer has to explicitly write the operations.
This also leads to the side effect that in Rust the explicit bitwise operations are type-safe
in the sense that it is ensured that the correct bit operations are performed on the intended
integer types.
