---
title: Ioctls
date: '2024-04-02'
categories:
  - Rust
  - C
  - Linux Kernel
  - Kernel Driver
  - Ioctls
tags:
  - Rust
  - Linux Kernel
  - Kernel Driver
  - Ioctls
  - UAPI
---

[<img class="penguin" src="/static/img/rusty_penguin_9.jpeg" alt="Rusty penguin. Created by DALL·E 3." />](https://github.com/Rust-for-Linux/)

# Input/Output Controls

In computing, ioctl calls (input/output calls) are special function calls used
for operations that can not be done with regular file operations like read,
write, etc. They provide a general-purpose interface for sending control codes
to devices, set parameters, etc.

The calls are a type of system call. But while system calls like open, read,
write, etc can be applied to all files, ioctls perform device specific
configurations. For example, they can be used to set the baudrate of a serial
driver, set the interface and network mask for network drivers, or reserve space
for a file for file system drivers.

The ioctl numbers, which are part of the UAPI (User API), can be found in
various header files. General ioctl numbers are located in the header file
`include/uapi/asm-generic/ioctl.h`, network-related ioctl numbers in
`include/uapi/linux/sockios.h`, and file system related numbers are in
`include/uapi/linux/fs.h`.

The ioctl numbers are generated using a macro that takes several parameters: the
direction (read or write), a character that defines the type of the call, the
ioctl number itself, and the size of the data. This macro generates a unique
identifier for each ioctl command.

```
#define _IOC(dir,type,nr,size) \
	(((dir)  << _IOC_DIRSHIFT) | \
	 ((type) << _IOC_TYPESHIFT) | \
	 ((nr)   << _IOC_NRSHIFT) | \
	 ((size) << _IOC_SIZESHIFT))
```

See for example the usage of this define in `include/uapi/linux/usbdevice_fs.h`.

```
#define USBDEVFS_CONNINFO_EX(len)  _IOC(_IOC_READ, 'U', 32, len)
```

The equivalent implementation in rust can be found in the file
`rust/kernel/ioctl.rs`.

```
#[inline(always)]
const fn _IOC(dir: u32, ty: u32, nr: u32, size: usize) -> u32 {
    build_assert!(dir <= uapi::_IOC_DIRMASK);
    build_assert!(ty <= uapi::_IOC_TYPEMASK);
    build_assert!(nr <= uapi::_IOC_NRMASK);
    build_assert!(size <= (uapi::_IOC_SIZEMASK as usize));

    (dir << uapi::_IOC_DIRSHIFT)
        | (ty << uapi::_IOC_TYPESHIFT)
        | (nr << uapi::_IOC_NRSHIFT)
        | ((size as u32) << uapi::_IOC_SIZESHIFT)
}
```

These identifiers are then used in switch-case statements to handle
device-specific commands. The `bindgen` tool in Rust generates pub const numbers
from these identifiers, allowing them to be used in a type-safe way in Rust
code.

For example, the `SIOCSIFNETMASK` ioctl number is defined in the
`include/uapi/linux/sockios.h` file and is represented in Rust as `pub const
SIOCSIFNETMASK: u32 = 35100;` in the `rust/uapi/uapi_generated.rs` and
`rust/bindings/bindings_generated.rs` files.

```
#define SIOCSIFNETMASK	0x891c		/* set network PA mask		*/
```

The `_IOC` function in Rust takes four arguments, three of which are of type u32
and one of usize, and returns an u32 value. What follows is the `build_assert!`
Rust macro, which enforces at compile time that the arguments passed to this
function do not surpass the maximum values. Furthermore, because all arguments
that are passed have types, the compiler can secure type safety of the function
arguments, while in the C macro, any value is accepted, which can lead to very
subtle errors. Lastly, the unique ioctl number is calculated in the same way it
is calculated in C, and the `_IOC` function returns this number.

The C header file `include/uapi/asm-generic/ioctl.h` provides also a way to
decode the ioctl numbers.

```
/* used to decode ioctl numbers.. */
#define _IOC_DIR(nr)		(((nr) >> _IOC_DIRSHIFT) & _IOC_DIRMASK)
#define _IOC_TYPE(nr)		(((nr) >> _IOC_TYPESHIFT) & _IOC_TYPEMASK)
#define _IOC_NR(nr)			(((nr) >> _IOC_NRSHIFT) & _IOC_NRMASK)
#define _IOC_SIZE(nr)		(((nr) >> _IOC_SIZESHIFT) & _IOC_SIZEMASK)
```

This same functionality is implemented in the Rust code as well, again located
in the file `rust/kernel/ioctl.rs`.  Those defines are again transcoded in
inherently type safe manner with functions instead of defines, which enforce a
type check at compile time. Other code can use those constant functions to
retrieve the data size with `_IOC_SIZE` or find out the direction (`_IOC_WRITE`
or `_IOC_READ`), which can then be used in the functions treating the ioctl
call.

```
/// Get the ioctl direction from an ioctl number.
pub const fn _IOC_DIR(nr: u32) -> u32 {
    (nr >> uapi::_IOC_DIRSHIFT) & uapi::_IOC_DIRMASK
}

/// Get the ioctl type from an ioctl number.
pub const fn _IOC_TYPE(nr: u32) -> u32 {
    (nr >> uapi::_IOC_TYPESHIFT) & uapi::_IOC_TYPEMASK
}

/// Get the ioctl number from an ioctl number.
pub const fn _IOC_NR(nr: u32) -> u32 {
    (nr >> uapi::_IOC_NRSHIFT) & uapi::_IOC_NRMASK
}

/// Get the ioctl size from an ioctl number.
pub const fn _IOC_SIZE(nr: u32) -> usize {
    ((nr >> uapi::_IOC_SIZESHIFT) & uapi::_IOC_SIZEMASK) as usize
}
```

To put it into a nutshell, in Rust, the ioctl functionality is implemented in a
type-safe manner using functions instead of macros. These functions enforce type
checks at compile time, which can help prevent subtle bugs that might occur if
any value were accepted.  This approach provides the benefits of type safety and
compile-time checks, making the code more robust and easier to maintain.
